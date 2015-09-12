---
layout: post
title: "KVM への FreeBSD11 インストール"
date: 2014-09-27 03:52
comments: true
categories: FreeBSD KVM virtualization
---

Debian 7.6 の KVM 上に FreeBSD11 Current をインストールした。備忘のため、ここに記録を残しておく。他環境で似たことをする時のために、なるべく KVM 固有の使い方はせずに virsh(libvirt) を利用して作業を行った。

## FreeBSD11 ISO イメージの取得

[ftp://ftp.freebsd.org/pub/FreeBSD/snapshots/amd64/amd64/ISO-IMAGES/11.0/](ftp://ftp.freebsd.org/pub/FreeBSD/snapshots/amd64/amd64/ISO-IMAGES/11.0/)

上記 URL からとりあえず現在最新の FreeBSD-11.0-CURRENT-amd64-20140918-r271779-bootonly.iso.xz を入手。

解凍
	% unxz FreeBSD-11.0-CURRENT-amd64-20140918-r271779-bootonly.iso.xz

tmp 以下に iso を展開

    % mkdir tmp
    % bsdtar -C tmp -pxvf FreeBSD-11.0-CURRENT-amd64-20140918-r271779-bootonly.iso

loader.conf に console 出力用のオプションを追加。

	% echo 'console="comconsole"' >> tmp/boot/loader.conf

元々 /boot/loader.conf というファイルは存在していなかったがとりあえず追記する形にした。これができたら iso に戻して展開したファイルを削除する。

    % genisoimage --no-emul-boot -R -J -b boot/cdboot -v -o	FreeBSD-11.0-CURRENT-amd64-serial.iso tmp
    % rm -rf tmp

## host 上にディスクイメージを作成

virsh では storage pool を用意してそこからディスクイメージ(volume)を作成する。今回は既存の storage pool として kvm_images があったので、そこから作ることにした。なければ "virsh pool-create-as" などで storage pool を作成しておく必要がある。詳しくは [http://libvirt.org/storage.html](http://libvirt.org/storage.html) を参照。

      % sudo virsh vol-create-as kvm_images FreeBSD11-CURRENT.qcow2 50G --format qcow2

FreeBSD11 を日常的に使う予定はないので、容量節約を第一に qcow2 を選択。qcow2 は実際に使用している分しか領域を消費しない上、copy-on-wirte、つまり、コピーしても何らかの書き込みが発生するまでは本当のコピーは発生しないという機能を有していることから使用するディスク領域を少なくできるが、領域拡大を含むディスクへの書き込みが非常に遅いのが特徴。
今回の目的ではディスク容量は最大 50GB もあれば充分。

## インストール

	% sudo virt-install --connect=qemu:///system --name=FreeBSD11-CURRENT --ram=4096 --vcpus 2 \
	--disk vol=kvm_images/FreeBSD11-CURRENT.qcow2,bus=virtio,driver_name=qemu,cache=none,io=threads \
	--network=bridge=br0,model=virtio \
	--graphics none  --noautoconsole \
	--cdrom=FreeBSD-11.0-CURRENT-amd64-serial.iso
	% sudo virsh console FreeBSD11-CURRENT

メモリ 4GB、 CPU 2個、HDD に先程作ったディスクイメージ、ネットワークは以前から作成していた br0、インストール元は iso。ディスクとネットワークには virtio を使用。以降 virsh からは --name オプションに与えた "FreeBSD11-CURRENT" でこの OS を指定する。

guest 側のコンソールに接続後、何だかマウントに失敗していたので、guest 側からマニュアルでマウント。

	mountroot> cd9660:cd0

続いてコンソールのタイプを聞かれた。

	Please choose the appropriate terminal type for your system.
	Common console types are:
	   ansi     Standard ANSI terminal
	   vt100    VT100 or compatible terminal
	   xterm    xterm terminal emulator (or compatible)
	   cons25w  cons25w terminal

	Console type [vt100]: 

ここはデフォルトの vt100 のまま。後は普通にインストールするだけ。sshd は使えるようにしておく。

インストールが終わったら再起動。

	% sudo virsh start FreeBSD11-CURRENT

起動後は ssh で接続。もちろん /boot/loader.conf に console="comconsole" を追加しておけば

	 % sudo virsh console FreeBSD11-CURRENT

でコンソールに接続できる。

後は通常の FreeBSD の操作と変わらない。大体のことは [FreeBSD Handbook](https://www.freebsd.org/doc/handbook/) に書いてある。
