---
layout: post
title: "The Design and Implementation of the FreeBSD Operating System Second Edition Chapter 3.1"
date: 2014-10-05 15:23
comments: true
categories: FreeBSD book
---

"The design and implementation of FreeBSD Operating System Second Edition" を読み始めた。内容理解の確認を兼ねてまとめたことを書いておく。あくまで個人的理解のために大雑把にまとめただけなので、色々間違っているだろうし、これだけでは何が書いてあるのかさっぱりかもしれない。

- FreeBSD カーネルはユーザプロセスに対するサービスプロバイダ
	- ユーザプロセスはシステムコールを介してサービスを受ける
	- カーネルモードで動くプロセスとして実装されているものもある
		- プロセススケジューリング
		- メモリ管理とか
- System Process
  	- p58. の Table3.1 に起動時に作られてずっと動いている大事なプロセスを挙げてる
	   	 - これらはカーネルプロセスといってカーネルモードで動く
	- カーネルプロセスの後にユーザモードで動く最初のプロセスを作る
		- init プロセス
			- プロセス 1
			- 管理タスクを行う
				- 端末毎に getty プロセスを作ったり
				- orphand process の exit コードを集めたり
			- ユーザモードのプロセスなので、カーネルの外で動く
- System Entry
  	 - カーネル内に入るには大きく 3 種類の方法がある
		- Hardware interrupt
			- 外部イベント起因
				- I/O とか
			- 非同期的に起こる
			- 実行中のプロセスとは無関係かも
		- Hardware trap
		  	- 同期的、非同期的両方起こる
			- 実行中のプロセスに関係する
				- 0 除算とか
		- Software-initiated trap
		  	- イベントをなるべく早くスケジューリング強制するのに使用
				- プロセスの再スケジューリング
				- ネットワーク処理
			- システムコールは Software-initiated trap の一種
			  	- Hardware trap を引き起こす命令を使用
- Run-Time Organization
	- カーネルは論理的に次の 2 つに分けられる
		- top half
			- システムコールやトラップに対応
			- 特権モードで実行
				- カーネルのデータ構造とユーザプロセスのコンテキスト双方にアクセス
					- コンテキスト
						- process structure
							- プロセスがスワップアウトしている時にも必要な情報
								- PID
								- discriptors
								- メモリマップ などなど
						- thread structure
							 - スワップアウトしている時には不要な情報
								- hardware thread state block(TSB)
								- カーネルスタック
								- デバッグやコアダンプ用の情報
		- bottom half
			 - hardware 割り込みに対して実行されるルーチン
			 	 - 割り込みソースに対して同期的
				 - top half に対しては非同期的
				 	- top half とはデータ構造を通してお話しする
	- カーネルは top half で実行中のユーザプロセスに対してたまにしか preempt しない
		- real-time プロセス
		- bottm half への割り込み には preempt する
	- システムリソースの共有ではプロセスは協調する
		- メモリとかディスクとか
	- top half と bottom half もシステムオペレーションで一緒に働く
		- I/O とか
			- top half が I/O オペレーションを開始する
				 - CPU を手放す
				 - リクエストを出したプロセスは sleep
			- I/O が完了したら bottom half が通知
- Entry to the Kernel
	- trap なり interrupt なりでカーネルに入る
		- 現在の machine state を保存
			 - program counter
			 - user stack pointer
			 - 汎用レジスタ
			 - processor status long word
	- trap や システムコールは次のイベントを引き起こす
		- カーネルモードに移行
			- スタックの参照にプロセス毎のカーネルスタックを使用
			- 特権命令使用可
		- カーネルスタックに以下のものを保存
			- program counter
			- processor status long word
			- trap の種類に関する情報
		- アセンブリ言語で書かれたルーチンで、hardware では保存されないものを保存
			- 汎用レジスタ
			- ユーザスタックポインタ
	- state を保存したらカーネルは C 言語のルーチンを呼ぶ
	- カーネルに入るハンドラには大きく 3 種類ある
		1. システムコール: syscal()
			- システムコール番号
			- exception frame
		2. hardware trap、システムコール以外の software-initiated trap: trap()
			- trap のタイプ
			- trap に関する浮動小数点と virtual address の情報
			- exception frame
		3. haredware interrupt: デバイスドライバの割り込みハンドラ
			- unit number
- Return from the Kernel
	 - やる事が終わったらユーザプロセスの state を復帰して制御を戻す
	 	- カーネルに入る時の逆
