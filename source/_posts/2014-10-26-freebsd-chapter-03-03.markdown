---
layout: post
title: The Design and Implementation of the FreeBSD Operating System Second Edition Chapter 3.3
date: 2014-10-26 13:21:57 +0900
comments: true
categories: book FreeBSD
---

Traps and Interrupts

- Trap
	- プロセスに同期して起こる
		- システムコールと同じ
	- 通常は意図しないエラーが原因
		- 0 除算
		- 不正ポインタアクセスとか
		- プロセスは意識して対処する必要あり
	- ページフォルトでも起こる
		- プロセスは気付かない
	- trap handler はシステムコールハンドラと似てる
		- process state を save
		- trap type を決めて signal を post するなり pagein を起こすなり
		- pending signal と高優先度のプロセスをチェック
		- handler から抜ける
			- 返戻値なし
- I/O Device Interrupts
	- I/O や他のデバイスからの割り込みは interrupt routine で扱う
		- カーネルのアドレス空間
		- 扱うのは
			- console terminal interface
			- 1 つ以上の時計
			- software-initiated の割り込みとか
	- 割り込みは非同期的に起こる
		- サービスの要求対象と今動いているプロセスは違うかも
			- 既に存在してないかも
			- 次に動く時に operation の終了が通知される
		- machine state を save しないといけないのはシステムコールや trap と同じ
	- Device-interrupt handler は要求に応じてのみ走る
		- device driver ごとに thread context を作成
			- interrupt handler は前に走っていた interrupt handler にはアクセスできない
			- 自身専用のスタックを持つ
	- 割り込みはリソース待ちでブロックするかも
		- ブロック中は他のイベントで invoke しない
			- 割り込みを lost しないようにほとんどの handler は sleep せずに完了まで走る
	- interrupt handler は top half からは決して起こらない
		- 必要な情報は top half と共有するデータ構造から得る
			- 一般的には global work queue
		- top half に渡す情報も同様
- Software Interrupts
	- 低 priority で早くやる必要のない処理を行う
		- 受けとったパケットをプロセスに渡すところとか
			- パケットを受けとるのは高速に処理しないといけないので hardware interrupt でやる
	- 高 priority の割り込みが より低い priority で処理する work の queue を作る
	- それぞれの software interrupt は process context を持つ
	- device driver より低く、ユーザプロセスよりは高い priority