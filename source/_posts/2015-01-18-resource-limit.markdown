---
layout: post
title: "FreeBSD におけるリソース制限"
date: 2015-01-18 10:47:30 +0900
comments: true
categories: freebsd
---

	% uname -a
	FreeBSD freebsd11 11.0-CURRENT FreeBSD 11.0-CURRENT #3 r272236: Sat Nov  1 17:27:44 JST 2014     root@freebsd11:/usr/obj/usr/src/sys/GENERIC  amd64


リソースに制限をかけるには setrlimit システムコールを使用する。

	setrlimit(int resource, const struct rlimit *rlp);

ここで各引数は

- resource: 制限をかけるリソース
- rlp: 制限

となっている。第二引数の struct rlimit は以下の通りで、 soft limit と hard limit を同時に設定する。

	struct rlimit {
		rlim_t  rlim_cur;       /* current (soft) limit */
		rlim_t  rlim_max;       /* maximum value for rlim_cur */
	};

ここでは rlim\_max が hard limit にあたる。例えば、CPU 時間に制限をかける場合、

``` c
    struct rlimit lim = {
        .rlim_cur = 2,
        .rlim_max = 15
	};
		
    if (setrlimit(RLIMIT_CPU, &lim) < 0) {
        perror("setrlimit() failed: ");
        exit(1);
    }
```

こんな感じで使うと soft limit が 2 秒、hard limit が 15 秒になる。ここで super user 以外で rlim_max を従来の hard limit より大きくしようとすると error になるので注意。

このようにして設定した制限がカーネル内部でどのように扱われているのかを見るために、まずは setrlimit のソースを調べる。sys\_setrlimit() は kern\_setrlimit() を呼び kern\_setrlimit() は更に、kern\_proc\_setrlimit() を呼んでいてこの kern\_proc\_setrlimit() が処理の本体になっている。

``` c kern/kern_resource.c start:661
int
kern_proc_setrlimit(struct thread *td, struct proc *p, u_int which,
    struct rlimit *limp)
```

which と limp は setrlimit に渡した引数で、td, p は制限をかける対象となるスレッド、プロセス構造体。一通り引数のチェック後、実際の設定を行う。

``` c kern/kern_resource.c start:706
        switch (which) {

        case RLIMIT_CPU:
                if (limp->rlim_cur != RLIM_INFINITY &&
                    p->p_cpulimit == RLIM_INFINITY)
                        callout_reset_sbt(&p->p_limco, SBT_1S, 0,
                            lim_cb, p, C_PREL(1));
                p->p_cpulimit = limp->rlim_cur;
                break;
        case RLIMIT_DATA:
                if (limp->rlim_cur > maxdsiz)
                        limp->rlim_cur = maxdsiz;
                if (limp->rlim_max > maxdsiz)
                        limp->rlim_max = maxdsiz;
                break;

        case RLIMIT_STACK:
                if (limp->rlim_cur > maxssiz)
                        limp->rlim_cur = maxssiz;
                if (limp->rlim_max > maxssiz)
                        limp->rlim_max = maxssiz;
                oldssiz = *alimp;
                if (p->p_sysent->sv_fixlimit != NULL)
                        p->p_sysent->sv_fixlimit(&oldssiz,
                            RLIMIT_STACK);
                break;

        case RLIMIT_NOFILE:
                if (limp->rlim_cur > maxfilesperproc)
                        limp->rlim_cur = maxfilesperproc;
                if (limp->rlim_max > maxfilesperproc)
                        limp->rlim_max = maxfilesperproc;
                break;

        case RLIMIT_NPROC:
                if (limp->rlim_cur > maxprocperuid)
                        limp->rlim_cur = maxprocperuid;
                if (limp->rlim_max > maxprocperuid)
                        limp->rlim_max = maxprocperuid;
                if (limp->rlim_cur < 1)
                        limp->rlim_cur = 1;
                if (limp->rlim_max < 1)
                        limp->rlim_max = 1;
                break;
        }
```

一部のリソースに関しては clamp 処理で制限が一定の範囲内に収まるようにしている。ただ、CPU の場合は少し特殊で、setrlimt によって使用できる CPU 時間が無制限でなくなった時に lim\_cb() が 1 秒毎に呼ばれるようにしている。lim\_cb() で行っている処理は、

``` c kern/kern_resource.c start:619
static void
lim_cb(void *arg)
{
        struct rlimit rlim;
        struct thread *td;
        struct proc *p;

        p = arg;
        PROC_LOCK_ASSERT(p, MA_OWNED);
        /*
         * Check if the process exceeds its cpu resource allocation.  If
         * it reaches the max, arrange to kill the process in ast().
         */
        if (p->p_cpulimit == RLIM_INFINITY)
                return;
        PROC_SLOCK(p);
        FOREACH_THREAD_IN_PROC(p, td) {
                ruxagg(p, td);
        }
        PROC_SUNLOCK(p);
        if (p->p_rux.rux_runtime > p->p_cpulimit * cpu_tickrate()) {
                lim_rlimit(p, RLIMIT_CPU, &rlim);
                if (p->p_rux.rux_runtime >= rlim.rlim_max * cpu_tickrate()) {
                        killproc(p, "exceeded maximum CPU limit");
                } else {
                        if (p->p_cpulimit < rlim.rlim_max)
                                p->p_cpulimit += 5;
                        kern_psignal(p, SIGXCPU);
                }
        }
        if ((p->p_flag & P_WEXIT) == 0)
                callout_reset_sbt(&p->p_limco, SBT_1S, 0,
                    lim_cb, p, C_PREL(1));
}
```

となっている。p->p\_rux.rux\_runtime が実際の実行時間で、p->p\_cpulimit は最初のうちは setrlimit で設定した soft limit。p\_cpulimit は秒なので単位を合わせるために cpu\_tickrate() をかけている。soft limit を超えているなら、次に hard limit と比較を行う必要がある。lim\_rlim() は setrlimit で設定した値を rlim にコピーするための関数で、rlim.rlim\_max が hard limit になる。よって、実行時間が hard limit を超えたなら killproc() でプロセスを kill し、そうでないなら p\_cpulimit を 5 増やした上で SIGXCPU を送信する。つまり、この SIGXCPU を無視なりしたら、5 秒後に再び SIGXCPU が送られる。lim\_rlimit() で soft limit を取得せずに p\_cpulimit を使用したのは、soft limit そのものに影響を与えずに 5 秒後に再設定できるようにするため。

というわけでこのことを確めるために以下のようなプログラムを作成してみた。

``` c resource_limit.c
#include <sys/types.h>
#include <sys/time.h>
#include <sys/resource.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>

static time_t start_sec;

static void
cpu_cb(int sig)
{
        struct timeval tv;

        gettimeofday(&tv, NULL);
        printf("%ld\n", tv.tv_sec - start_sec);
};


int
main(int argc, char **argv)
{
        struct rlimit lim = {
                .rlim_cur = 2,
                .rlim_max = 15
        };
        struct timeval tv;
        int i;

        signal(SIGXCPU, cpu_cb);

        gettimeofday(&tv, NULL);
        start_sec = tv.tv_sec;
        if (setrlimit(RLIMIT_CPU, &lim) < 0) {
                perror("setrlimit() failed: ");
                exit(1);
        }

        for( ;; );

        /* never come here */
        return 0;
}
```
CPU 時間に対する soft limit を 2 秒、hard limit を 15 秒に設定した上で無限ループするだけのプログラムだが、SIGXCPU シグナルを処理するので、soft limit を超えても終了することはない。よって上述の通りなら、2, 7, 12 秒の時点で SIGXCPU シグナルが送信され、17 秒で kill されるはず。hard limit として設定した 15 秒でないのは、実行時間が p->p\_cpulimit を超えない限り hard limit との比較は行われず、12 秒の次に p->p\_cpulimit がとる値は 17 になるため。というわけで実際に実行してみたところ、

	% time ~/src/freebsd/a.out
	2
	7
	12
	zsh: killed     ./a.out
	./a.out  17.29s user 0.00s system 5775188% cpu 17.315 total

となった。ほぼ予想通り。

CPU 時間以外のリソースの制限方法はリソース毎に異なる。例としてファイル数 RLIMIT\_NOFILE などは fdalloc() 内で比較を行い、file descriptor の数が soft limit を超えていたら error を返す。

最後に、CPU とは異なるものの、RLIMIT\_STACK も特殊な制限の仕方をしており、virtual memory の protection を利用して制限を超えないようにしている。詳しくは 
kern\_proc\_setrlimit() の残りを読めば分かる。
