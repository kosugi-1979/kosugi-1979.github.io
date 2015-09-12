---
layout: post
title: "Jail + VIMAGE のセットアップ"
date: 2015-05-06 15:42:30 +0900
comments: true
categories: freebsd jail network virtualization
---

[前回](/blog/2015/05/05/jail)、jail を使うためのセットアップを行った。この状態ではネットワークスタックが host と共通なので、ネットワークがらみで遊んでみようとするとあまり使い勝手がよろしくない。そこで VIMAGE を有効にする。

とりあえず、前回作成した ``jail\_01'' と ``jail\_02'' という 2 つの jail を相互に接続することを目標とする。

## VIMAGE対応カーネルのビルド ##

VIMAGE はまだ GENERIC カーネルには含まれていない機能なので、使うにはカーネルをリビルドする必要がある。

まずはこれから作るカーネル用のコンフィグファイルを用意する。

	# cd /usr/src/sys/amd64/conf
	# cp ./GENERIC ./VIMAGE

作成した VIMAGE に以下の行を追加する。

	options         VIMAGE

カーネルをリビルド、インストールしてから再起動

	# cd /usr/src
	# make -j 4 buildkernel KERNCONF=VIMAGE
	# make installkernel KERNCONF=VIMAGE
	# reboot

## epair の作成 ##

epair(4) の man を読むと、virtual なクロスケーブルで繋がった Ethernet みたいな interface のペアとのこと。epair を使うには if_epair カーネルモジュールをロードしておかなければならない。ロードされているかは

% kldstat | grep if_epair

で確認できる。ロードされていなかった場合には kldload を使うか loader.conf で if_epair をロードしておくこと。

ifconfig を用いて epair を作るには

	# ifconfig epair<n> create

とする。``&lt;n&gt;'' には数字が入り、省略した場合には適当な数字が使われる。こうすると、epair&lt;n&gt;a と epair&lt;n&gt;b の 2 つの interface が作成され、双方が互いに接続されているように振る舞う。なお、削除する時にはどちらか片方を destroy すれば両方とも削除される。

	# ifconfig epair<n>a destroy

これを利用して jail のインスタンス同士を接続する。

	# ifconfig epair1 create

として作成した epair1a と epair1b をそれぞれ ``jail\_01'' と ``jail\_02'' に割り当ててやれば ethernet で相互に接続した状態を作ることができる。割り当ての仕方は後述する。

## vnet ##

jail で自身の virtual なネットワークスタックを有効にするには vnet パラメータに ``new'' を与える。

	vnet="new"

上記の記述を jail.conf に追加するか、jail のコマンドラインに加えてやれば良い。ただし、vnet を使っている時には ip4.addr パラメータなどで IP address を制限することはできなくなる。

このようにして vnet を有効にした jail に対して network interface を与えるために vnet.interface パラメータを使う。先程作成した epair1a を ``jail\_01'' に与えるには jail.conf で

	vnet.interface="epair1a"

のようにするか、ifconfig で

	# ifconfig epair1a vnet jail_01

とする。これで network interface が jail 内から見えるようになる。

この epair に IP address を割り当てるには exec.start を使って jail 内から ifconfig で行う。jail.conf に書くのであれば、

	exec.start = ifconfig epair1a inet 10.0.0.1 netmask 255.255.255.0 up

という形式になる。もちろん jail のコマンドラインから与えることもできるし、jail 起動後に jexec で実行することもできる。

以上のことを jail.conf に反映させると、

	exec.start = "ifconfig epair${if} inet ${ipv4addr} netmask 255.255.255.0 up";
	exec.start += "/bin/sh /etc/rc";
	exec.stop = "/bin/sh /etc/rc.shutdown";
	exec.clean;
	
	vnet="new";
	vnet.interface = "epair${if}";
	
	mount  = "/home/jail/base/bin /home/jail/${name}/bin nullfs ro 0 0";
	mount += "/home/jail/base/lib /home/jail/${name}/lib nullfs ro 0 0";
	mount += "/home/jail/base/libexec /home/jail/${name}/libexec nullfs ro 0 0";
	mount += "/home/jail/base/sbin /home/jail/${name}/sbin nullfs ro 0 0";
	mount += "/home/jail/base/etc /home/jail/${name}/etc nullfs rw 0 0";
	mount += "/home/jail/base/home /home/jail/${name}/home nullfs rw 0 0";
	mount += "/home/jail/base/usr /home/jail/${name}/usr nullfs rw 0 0";
	mount += "/home/jail/base/root /home/jail/${name}/root nullfs rw 0 0";
	mount += "/home/jail/base/var /home/jail/${name}/var nullfs rw 0 0";
	mount += "/usr/ports /home/jail/${name}/usr/ports nullfs ro 0 0";
	mount += "/home/jail/rw/distfiles /home/jail/${name}/usr/ports/distfiles nullfs rw 0 0";
	mount.devfs;
	devfs_ruleset="1000";
	allow.raw_sockets;
	
	path = "/home/jail/${name}";
	host.hostname = "$name";
	interface = "vtnet0";
	
	jail_01 {
        $if = 1a;
        $ipv4addr = 10.0.0.1/24;
	}
	
	jail_02 {
        $if = 1b;
        $ipv4addr = 10.0.0.2/24;
	}

となる。``${if}'' と ``${ipv4addr}'' はこちらで定義した変数で、jail インスタンス毎に異なる値をとる。上記設定ファイルを用いた場合のネットワーク構成は以下の通り。

![vimage network](/images/2015-05-06-01.svg)

rc.conf 変更後、jail を再起動した上で、``jail_01'' から ``jail_02'' に対して ping を飛ばすと、

	% sudo jexec jail_01 ping 10.0.0.2
	capability mode sandbox enabled
	PING 10.0.0.2 (10.0.0.2): 56 data bytes
	64 bytes from 10.0.0.2: icmp_seq=0 ttl=64 time=0.092 ms
	64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.142 ms
	64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.125 ms
	64 bytes from 10.0.0.2: icmp_seq=3 ttl=64 time=0.148 ms
	^C
	--- 10.0.0.2 ping statistics ---
	4 packets transmitted, 4 packets received, 0.0% packet loss
	round-trip min/avg/max/stddev = 0.092/0.127/0.148/0.022 ms

のように ping が通っているのがわかる。

個人的に気になるのは epair の作成を jail.conf の外側で行っており、jail 起動前に epair を作っておく必要があること。jail.conf は jail インスタンス毎に使用されるため、``jail_01'' と ``jail_02'' で共用する epair1 を作成したい時にはどうすれば良いのかが分からない。共通部で行うと 2 回同じコマンドが発行されてしまうし、``jail_01'' 固有部で行うと ``jail_02'' から先に起動した時に問題となる。設定関連は一か所にまとめておきたいのだが、jail.conf に頼らず自分でスクリプトを書くべきなのだろうか。

他にも何かやったら追記するかも。
