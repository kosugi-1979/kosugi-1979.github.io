---
layout: post
title: "The Design and Implementation of the FreeBSD Operating System Second Edition Chapter 12.2"
date: 2015-05-30 09:21:57 +0900
comments: true
categories: book freebsd
---

## Implementation Structure and Overview ##

* プロセス間通信はネットワークレイヤの上に位置する
    * データはアプリケーションからソケットレイヤを通してネットワークレイヤという流れ
        * またはその逆
    * ソケットレイヤの state は内部にカプセル化
    * プロトコル関連の state はプロトコル固有のデータ構造内で管理
    * ソケットレイヤ内ではソケットデータ構造がアクティビティの中心
        * ソケットによる抽象化のほとんどはソケットレイヤ内のルーチンで実装されている
            * ソケットレイヤのルーチンは *so* のプレフィックスが付いている
                * データ構造の操作
                * 非同期アクティビティ間の同期管理
