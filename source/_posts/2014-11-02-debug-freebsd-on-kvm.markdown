---
layout: post
title: "KVM 上の FreeBSD/amd64 を GDB でリモートデバッグ"
date: 2014-11-02 14:20:25 +0900
comments: true
categories: FreeBSD KVM
---

以前に [KVM に FreeBSD11 をインストールした](/blog/2014/09/27/freebsd11-on-kvm)のだけれど、この FreeBSD の動作を調べたくなったので、Debian GNU/Linux 7.6 で動いている host マシンからのリモートデバッグを試した。

## Target(FreeBSD) 側の作業

まずはデバッグ用にカーネルをリビルドする。基本的には [FreeBSD Handbook](https://www.freebsd.org/doc/handbook/kernelconfig.html) にある通りにすれば良い。リモートデバッグをする際、[FreeBSD Developers' Handbook](https://www.freebsd.org/doc/en/books/developers-handbook/kerneldebug-online-gdb.html) によると、コンフィグファイルに

	makeoptions DEBUG=-g

とデバッグ用のオプションを書いてカーネルをリビルドしておく必要があるとのことだが、GENERIC のコンフィグファイルには最初からこの記述があった。むしろ、このままカーネルをビルドすると最適化の "-O2" オプションが入ってしまうことの方が問題で、こちらは /etc/make.conf に

	COPTFLAGS= -O0 -pipe

を追加する。別に "-O2" でもデバッグが不可能なわけではないが、ソースとの対応がずれたりして面倒なことになりやすい。どうしても最適化する必要がないなら "-O0" でビルドすべき。

後は普通にリビルド、インストールするだけ。

## Host(Debian) 側の作業

Tareget の側でカーネルのリビルド、インストールが終わったら当然再起動するわけだけれど、その時に qemu に "-s" オプションを付けて起動する。このオプションを付けると localhost:1234 で gdbserver を open してくれるので、gdb でそこに接続してリモートデバッグを行う。なお、他のポートを使いたい場合には "-s" の代わりに "-gdb" オプションを用いる。詳しくは qemu の man を参照。

libvert を使っている環境で qemu にオプションを付けるには、設定用の xml(自分の環境では /etc/libvirt/qemu/FreeBSD11-CURRENT.xml) の domain タグを

	<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>

とした上で

	  <qemu:commandline>
	      <qemu:arg value='-s'/>
	  </qemu:commandline>

を追加する。通常のエディタで編集した場合には

	% sudo virsh define /etc/libvirt/qemu/FreeBSD11-CURRENT.xml

を忘れないよう注意。オプションを加えたらリビルドしたカーネルで FreeBSD を起動する。

	% sudo virsh start FreeBSD11-CURRENT

まだ Target の FreeBSD を shutdown していなければ `start` ではなく `reboot` にする。domain-id は適宜読み替えること。

続いてカーネルのソースとバイナリを取得する。方法は何でも良いが、今回は rsync を使用した。ダウンロード先は新規に作成した ~/src/freebsd11 というディレクトリ。

	% mkdir -p src/freebsd11
	% cd src/freebsd11
	% rsync -avz --rsh=ssh username@192.168.1.xxx:/usr/src/sys ./
	% rsync -avz --rsh=ssh username@192.168.1.xxx:/boot/kernel ./

username と target の IP アドレスは適宜読み替える。このままだとソースの場所が合わないので、

	% sudo ln -s ~/src/freebsd11/sys /usr/src/

として /usr/src/sys でソースにアクセスできるようにしておく。

最後に gdb の準備だが、今回は Linux 上で FreeBSD のデバッグを行う事になるので、クロスデバッグになる。そのため、gdb もそれに対応したものを用意する必要がある。幸い、Debian では gdb-multiarch なるものがあるので、それを利用させてもらう。

	% sudo aptitude install gdb-multiarch

これでリモートデバッグの準備は完了。

## 実際にデバッグしてみる

host 側で gdb-multiarch を使って target に接続する。先程の ~/src/freebsd11 に移動してから、以下を入力する。

	% gdb-multiarch kernel/kernel 
	(gdb) target remote localhost:1234

gdb からの出力も合わせると、

	% gdb-multiarch kernel/kernel 
	GNU gdb (GDB) 7.4.1-debian
	Copyright (C) 2012 Free Software Foundation, Inc.
	License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
	This is free software: you are free to change and redistribute it.
	There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
	and "show warranty" for details.
	This GDB was configured as "x86_64-linux-gnu".
	For bug reporting instructions, please see:
	<http://www.gnu.org/software/gdb/bugs/>...
	Reading symbols from /home/username/src/freebsd11/kernel/kernel...Reading symbols from /home/username/src/freebsd11/kernel/kernel.symbols...done.
	done.
	(gdb) target remote localhost:1234
	Remote debugging using localhost:1234
	acpi_cpu_c1 () at /usr/src/sys/amd64/acpica/acpi_machdep.c:95
	95      }
	(gdb) 

こんな感じ。後は通常のデバッグと同様に操作できる。なお、emacs の gdb モードも使えるので、そちらを使えば更に便利になる。

毎回 target への接続を行うのが面倒なら、~/src/freebsd11 以下に .gdbinit を作成し、

	file kernel/kernel
	target remote localhost:1234

とでも書いておけば良い。これで ~/src/freebsd11 から `gdb-multiarch` とするだけでデバッグを開始できる。

試しに前に調べた[システムコールの呼出し](/blog/2014/10/19/freebsd-systemcall)に関して見てみる。write システムコールは最終的に sys_write() を呼出しているはずなので、そこに breakpoint を仕掛ける。 

	(gdb) b sys_write
	Breakpoint 1 at 0xffffffff80ded614: file /usr/src/sys/kern/sys_generic.c, line 374.
	(gdb) c
	Continuing.

write は頻繁に呼ばれるシステムコールなので、target 側で少しでも操作しようとするとすぐに breakpoint に引っかかる。
	
	Breakpoint 1, sys_write (td=0xfffff80005406490, uap=0xfffffe00f6b48b98)
	    at /usr/src/sys/kern/sys_generic.c:374
	374             if (uap->nbyte > IOSIZE_MAX)

backtrace を見ると、

	(gdb) bt
	#0  sys_write (td=0xfffff80005406490, uap=0xfffffe00f6b48b98)
	    at /usr/src/sys/kern/sys_generic.c:374
	#1  0xffffffff8135a50e in syscallenter (td=0xfffff80005406490, 
	    sa=0xfffffe00f6b48b88)
	    at /usr/src/sys/amd64/amd64/../../kern/subr_syscall.c:133
	#2  0xffffffff81359ebf in amd64_syscall (td=0xfffff80005406490, traced=0)
	    at /usr/src/sys/amd64/amd64/trap.c:967
	#3  0xffffffff8132ed8b in Xfast_syscall ()
	    at /usr/src/sys/amd64/amd64/exception.S:390
	#4  0x0000000000000008 in ?? ()
	#5  0x00007fffffffe393 in ?? ()
	#6  0x0000000000000005 in ?? ()
	#7  0x0000000000000000 in ?? ()

と、カーネル内部に入ってからおおむね[前の記述](/blog/2014/10/19/freebsd-systemcall)通りに sys_write() が呼ばれていることがよく分かる。
