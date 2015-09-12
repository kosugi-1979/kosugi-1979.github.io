---
layout: post
title: FreeBSD/amd64 のシステムコール
date: 2014-10-19 16:17:12 +0900
comments: true
categories: FreeBSD
---
FreeBSD 11.0-CURRENT r272236 amd64 が対象

以下のような単純な例で FreeBSD/amd64 システムコール呼出しを見てみる。

	#include <sys/types.h>
	#include <unistd.h>

	int
	main(int argc, char **argv)
	{
		write(STDOUT_FILENO, "hello", 6);
		return 0;
	}

上記の例はプログラムとして完結した形にしているが、目下の所関心があるのは write() の部分のみ。man で write を調べてみると、

	man 2 write

write() は libc 内で定義されている関数とのこと。というわけで libc 内のシンボルを見てみる。

	nm -g /usr/lib/libc.a

上記コマンドの出力によると、write のシンボルは write.o にあるらしい。write.o ということは write.c なりのファイルが /usr/src/lib/libc 以下にありそうな気がするものの、見つからない。
というのも、write.o のソースは buildworld 時に自動的に作成されるからで、そのコードは /usr/obj/usr/src/lib/libc/write.S というアセンブリ言語のファイルになっている。どのように作成されるのかに興味がある人は /usr/src/lib/libc/Makefile から include されている /usr/src/lib/libc/sys/Makefile.inc を読めば分かる。

自分で buildworld をしたのなら、write.S が存在し、その中身は

	#include "compat.h"
	#include "SYS.h"
	RSYSCALL(write)
		.section .note.GNU-stack,"",%progbits

となっている。include を除けば 2 行だ。しかも .section の方は http://en.chys.info/2010/12/note-gnu-stack/ によると executable stack を必要としないことを linker に教えるためのものなので、意味を持つのは

	RSYSCALL(write)

の部分だけとなる。RSYSCALL は /usr/src/lib/libc/amd64/SYS.h で定義されているマクロで、

	#define RSYSCALL(name)  ENTRY(__sys_##name);
	                        WEAK_REFERENCE(__sys_##name, name);
	                        WEAK_REFERENCE(__sys_##name, _##name);
	                        mov $SYS_##name,%eax; KERNCALL;
	                        jb HIDENAME(cerror); ret;
	                        END(__sys_##name)

もっと詳しく見たければプリプロセッサを通せば良い。

	 cpp -I /usr/src/lib/libc/include -I /usr/src/lib/libc/amd64 /usr/obj/usr/src/lib/libc/write.S 

とはいえ、別にそこまでしなくても大体の雰囲気はつかめる。write の場合、まず \_\_sys_write というエントリを作成し、それへの weak reference として write と \_write を用意する。そういえば上記の nm の出力結果は

	0000000000000000 T __sys_write
	0000000000000000 W _write
	0000000000000000 W write

となっており、記述と合致する。

続いて $SYS_write を eax レジスタに入れて KRENCALL するわけだが、SYS_write はシステムコールの番号で、/usr/src/sys/sys/syscall.h に write も含めて記載されている。それによると write は 4 とのこと。なお、"$SYS_write" と '$' で始まっているのは immediate value だから。

KERNCALL も RSYSCALL 同様 /usr/src/lib/libc/amd64/SYS.h で定義されているマクロで、

	 #define KERNCALL        movq %rcx, %r10; syscall

rcx を r10 にコビーした後で syscall を呼んでいる。

syscall という名前からしてここがシステムコールの肝っぽいので、ひとまずここの処理を追うのは後回しにし、次の命令に進む。次は jb となっており、この命令は carry bit が 1 なら operand で指定した場所に飛ぶというもの。システムコール失敗時には carry bit が 1 になっているので、失敗時の処理が HIDENAME(cerror) にあるということになる。  
詳しく知りたいなら、/usr/src/lib/libc/amd64/sys/cerror.S と /usr/src/lib/libc/sys/__error.c を読むと分かる。簡単に説明するとエラーコードを errno に入れ、 rax と rdx を -1 にして関数(今回は write())から return する。 rax は関数の返戻値になるので、システムコールが失敗したら呼出し側に -1 が返ってくる。

syscall 周辺のことは大体分かったので、いよいよ本体に挑む。この命令に関しては http://www.marbacka.net/asm64/arkiv/int2e_sysenter_syscall.html に詳しく、それによると Model Specific Register(MSR) の LSTAR にあるアドレスに飛ぶとのこと。LSTAR への書き込みを探してみると、/usr/src/sys/amd64/amd64/machdep.c と /usr/src/sys/amd64/amd64/mp_machdep.c に見つかるのだが、内容は両方とも同じで、

	wrmsr(MSR_LSTAR, (u_int64_t)IDTVEC(fast_syscall));

となっており、syscall は fast_syscall に飛ぶようだ。fast_syscall は /usr/src/sys/amd64/amd64/exception.S で定義されている。

いよいよここから先はカーネルモードの世界に入る。fast_syscall は長いのでここに載せることはしないが、スタックを切り替えた上で引数をそのスタックに乗せて、amd64_syscall を呼んでいる。ちなみに引数は 6 個まで。スタックに乗せる際には後で C 言語のコードからアクセスしやすいように trapframe 構造体の作りに合わせている。trapframe に関しては /usr/src/sys/x86/include/frame.h を参照。なお、同一ファイルに i386 用と amd64 用の両方が定義されているので注意。

amd64_syscall() は /usr/src/sys/amd64/amd64/trap.c で定義されている。また、ここからは再び C 言語で書かれているコードにもどる。amd64_sycall() は

	void        amd64_syscall(struct thread *td, int traced);

で、第一引数が struct thread へのポインタになっている。/usr/src/sys/sys/proc.h を見ると、struct thread はさらにメンバとして trapframe へのポインタを持っていて、先程の fast_syscall 内でスタック上に配置した trapframe はここからアクセスできるようにしている。

amd64_syscall からはアーキテクチャ非依存の /usr/src/sys/kern/subr_syscall.c 内 syscallenter() を呼ぶ。syscallenter() はトレース用のコードなどがあって結構長めの関数になっているのだが、引数を fetch してからシステムコールを呼出す。  
fetch は /usr/src/sys/amd64/amd64/trap.c の cpu_fetch_syscall_args() で行っている。ちなみに amd64_syscall() 内で自動変数としてスタック上に確保している領域に fetch しているので、syscallenter() から return 後にも参照可能なようにしている。  
この関数は他にも

	sa->callp = &p->p_sysent->sv_table[sa->code];

でシステムコール本体の fetch もしている、sv_table は /usr/src/sys/kern/init_sysent.c 内の sysent で、write が 4 番目の要素になっていることが確認できる。

システムコール呼出しは syscallenter() 内で

	error = (sa->callp->sy_call)(td, sa->args);

という形になっているので、さっきの sysent と合わせると sys_write() がシステムコール本体だと分かる。sys_write() は /usr/src/sys/kern/sys_generic.c で定義されている。

最後の方は駆け足気味になってしまったが、システムコール呼出しの共通部分はここまで。ここから先は write() 固有の処理になり、「システムコール呼出しを見る」という今回の目的からは外れるのでここでは扱わない。