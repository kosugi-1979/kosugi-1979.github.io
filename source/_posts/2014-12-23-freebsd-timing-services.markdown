---
layout: post
title: "FreeBSD/amd64 の時刻調整処理"
date: 2014-12-23 11:26:34 +0900
comments: true
categories: freebsd
---

	% uname -a
	FreeBSD freebsd11 11.0-CURRENT FreeBSD 11.0-CURRENT #3 r272236: Sat Nov  1 17:27:44 JST 2014     root@freebsd11:/usr/obj/usr/src/sys/GENERIC  amd64


## bintime 構造体

カーネル内部では struct bintime という構造体で時間を表していることが多い。

``` c sys/time.h start:53
struct bintime {
        time_t  sec;
        uint64_t frac;
};
```

ここで sec は秒、frac は秒未満の値で 2^64 を 1 秒とするもの。sys/time.h にはこの struct bintime を操作するための関数も一緒に記述されているので、それを読めば大体のことは分かるはず。

## timehands 構造体

kern/kern_tc.c 内では struct timehands という構造体が定義されている。

``` c kern/kern_tc.c start:65
struct timehands {
        /* These fields must be initialized by the driver. */
        struct timecounter      *th_counter;
        int64_t                 th_adjustment;
        uint64_t                th_scale;
        u_int                   th_offset_count;
        struct bintime          th_offset;
        struct timeval          th_microtime;
        struct timespec         th_nanotime;
        /* Fields not to be copied in tc_windup start with th_generation. */
        volatile u_int          th_generation;
        struct timehands        *th_next;
};
```

使用する timecounter を th\_counter に保持し、

- th\_offset\_count は timecounter のカウント数で tc\_windup() が呼ばれる度に th\_counter を用いで更新される
- カウントが th\_counter->tc\_frequency に達すると 1 秒
- th\_offset は boottime からのオフセット
- th\_microtime、th\_nanotime はそれぞれ struct timeval、struct timespec 型で時刻を表したもの

なお、amd64 では timecounter には ACPI の HPET が使用されていた。dev/acpica/acpi_hpet.c の hpet\_get\_timecount() などを参照。

最後のメンバ th\_next を見れば予想がつくが、struct timehands は linked list になっている。要素数は 10 で、 th[0-9] という名前で定義されており、初期化の段階で添字順ににつなぎ、最後の th9 は th0 につながってリング状になっている。このリングの中で現在操作の対象といるものを "timehands" で指す。

``` c kern/kern_tc.c start:65
static struct timehands th0;
static struct timehands th9 = { NULL, 0, 0, 0, {0, 0}, {0, 0}, {0, 0}, 0, &th0};
static struct timehands th8 = { NULL, 0, 0, 0, {0, 0}, {0, 0}, {0, 0}, 0, &th9};
static struct timehands th7 = { NULL, 0, 0, 0, {0, 0}, {0, 0}, {0, 0}, 0, &th8};
static struct timehands th6 = { NULL, 0, 0, 0, {0, 0}, {0, 0}, {0, 0}, 0, &th7};
static struct timehands th5 = { NULL, 0, 0, 0, {0, 0}, {0, 0}, {0, 0}, 0, &th6};
static struct timehands th4 = { NULL, 0, 0, 0, {0, 0}, {0, 0}, {0, 0}, 0, &th5};
static struct timehands th3 = { NULL, 0, 0, 0, {0, 0}, {0, 0}, {0, 0}, 0, &th4};
static struct timehands th2 = { NULL, 0, 0, 0, {0, 0}, {0, 0}, {0, 0}, 0, &th3};
static struct timehands th1 = { NULL, 0, 0, 0, {0, 0}, {0, 0}, {0, 0}, 0, &th2};
static struct timehands th0 = {
        &dummy_timecounter,
        0,
        (uint64_t)-1 / 1000000,
        0,
        {1, 0},
        {0, 0},
        {0, 0},
        1,
        &th1
};

static struct timehands *volatile timehands = &th0;
```
## adjtime システムコール

sys_adjtime() のコードは、

``` c kern/kern_ntptime.c start:945
int
sys_adjtime(struct thread *td, struct adjtime_args *uap)
{
        struct timeval delta, olddelta, *deltap;
        int error;

        if (uap->delta) {
                error = copyin(uap->delta, &delta, sizeof(delta));
                if (error)
                        return (error);
                deltap = &delta;
        } else
                deltap = NULL;
        error = kern_adjtime(td, deltap, &olddelta);
        if (uap->olddelta && error == 0)
                error = copyout(&olddelta, uap->olddelta, sizeof(olddelta));
        return (error);
}
```
となっており、ここでは引数の処理のみを行い、本体は kern_adjtime() で行っている。kern_adjtime() は

``` c kern/kern_ntptime.c start:964
int
kern_adjtime(struct thread *td, struct timeval *delta, struct timeval *olddelta)
{
        struct timeval atv;
        int error;

        mtx_lock(&Giant);
        if (olddelta) {
                atv.tv_sec = time_adjtime / 1000000;
                atv.tv_usec = time_adjtime % 1000000;
                if (atv.tv_usec < 0) {
                        atv.tv_usec += 1000000;
                        atv.tv_sec--;
                }
                *olddelta = atv;
        }
        if (delta) {
                if ((error = priv_check(td, PRIV_ADJTIME))) {
                        mtx_unlock(&Giant);
                        return (error);
                }
                time_adjtime = (int64_t)delta->tv_sec * 1000000 +
                    delta->tv_usec;
        }
        mtx_unlock(&Giant);
        return (0);
}
```

で、time\_adjtime に delta を代入している。delta が struct timeval 型なのに対して time\_adjtime は int64_t 型で mircoseconds 単位の値になっているため、変換処理が入っているものの、処理内容自体は素直。

## 時刻の調整

adjtime では時刻の差分を記録するのみと分かったので、次は time\_adjtime をどこで使用しているかを調べる。すると、ntp\_update\_second() という関数内に

``` c kern/kern_ntptime.c start:582
        /*
         * Apply any correction from adjtime(2).  If more than one second
         * off we slew at a rate of 5ms/s (5000 PPM) else 500us/s (500PPM)
         * until the last second is slewed the final < 500 usecs.
         */
        if (time_adjtime != 0) {
                if (time_adjtime > 1000000)
                        tickrate = 5000;
                else if (time_adjtime < -1000000)
                        tickrate = -5000;
                else if (time_adjtime > 500)
                        tickrate = 500;
                else if (time_adjtime < -500)
                        tickrate = -500;
                else
                        tickrate = time_adjtime;
                time_adjtime -= tickrate;
                L_LINT(ftemp, tickrate * 1000);
                L_ADD(time_adj, ftemp);
        }
        *adjustment = time_adj;
```

という個所が見つかる。time\_adjtime != 0 というのは時刻調整の必要があるという意味で、調整量に応じて tickrate を変えているようだ。1sec よりも大きければ 5000 、501 microsec から 1sec までは 500、500 以下は調整量をそのまま使用する。L\_LINT は

``` c kern/kern_ntptime.c start:76
#define L_LINT(v, a)    ((v) = (int64_t)(a) << 32)
```
なので、ftemp の上位 32bit に tickrate * 1000、入れると解釈できる。これと time\_adj の和が新しい time\_adj になる。この time\_adj の値がそのまま代入されている adjustment は ntp\_update\_second() の引数で、caller 側に time\_adj を渡している。
当の caller はというと、tc_windup() がそれに当たる。

``` c kern/kern_tc.c start:1305
        /*
         * Deal with NTP second processing.  The for loop normally
         * iterates at most once, but in extreme situations it might
         * keep NTP sane if timeouts are not run for several seconds.
         * At boot, the time step can be large when the TOD hardware
         * has been read, so on really large steps, we call
         * ntp_update_second only twice.  We need to call it twice in
         * case we missed a leap second.
         */
        bt = th->th_offset;
        bintime_add(&bt, &boottimebin);
        i = bt.sec - tho->th_microtime.tv_sec;
        if (i > LARGE_STEP)
                i = 2;
        for (; i > 0; i--) {
                t = bt.sec;
                ntp_update_second(&th->th_adjustment, &bt.sec);
                if (bt.sec != t)
                        boottimebin.sec += bt.sec - t;
        }

```

この処理に来る前に、tc\_offset は更新されており、それに bttotimebin を足すことで bt は現在時刻を指すようになっている。また、tho は現在の timehands 構造体なので、LARGE\_STEP を超えない限りは秒の差分回 ntp\_update\_second() を呼ぶことになる。

ntp\_update\_second() の第二引数は、コメントによるとうるう秒の分を調整している。なので、通常は bt.sec == t になるが、そうならなかった場合には boottimebin にもそれを反映させている。そして th\_adjustment は何に用いるかというと、scale factor、つまりは th->th\_scale を更新する際に使用する。th は tho の次の timehands 構造体で、次回 tc\_windup() を呼んだ時にはこの th が tho になる。scale factor の計算に関しては tc\_windup() 内に詳しいコメントがあるので、それを参照。

scale factor は bintime の frac を進める際に使用され、これの大小で時間の進みが早くなったり遅くなったりする。つまり、時刻調整はこの frac の進み方を変えることで実現している。

それにしても、"The Design and Implementation of the FreeBSD Operating System Second Edition" では時刻の進みを 10% 早めるか遅くするかで時刻調整を行うように書いてあったが、ソースを見る限りではとてもそのような処理をしているようには見えない。何か重大な読み違いでもしているのだろうか。

## Feed Forward clock に関して

今回ソースを読んでいると、色々な場所で #ifdef FFCLOCK 〜 #endif に囲まれているコードを見かけた。このコードは conf/NOTES によると、

	# Enable support for generic feed-forward clocks in the kernel.
	# The feed-forward clock support is an alternative to the feedback oriented
	# ntpd/system clock approach, and is to be used with a feed-forward
	# synchronization algorithm such as the RADclock:
	# More info here: http://www.synclab.org/radclock
	
	options         FFCLOCK

であり、カーネルビルド時にコンフィグファイルで上記の options を加えることで従来の ntpd/system clock のような feedback アプローチと異なる feed-forward clock を有効にするものらしい。feed-forward clock が用いているアルゴリズムは上記コメント内の URL [RADClock](http://www.synclab.org/radclock) を参照とのこと。

このあたりのことも "The Design and Implementation of the FreeBSD Operating System Second Edition" では全く触れられていない。実装が変わっているにもかかわらず、First Edition、もしくは 4.4BSD からの記述を更新できていないのだろうか。また、そうした部分が他にもあるのではないかと少し心配になる。
