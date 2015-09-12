---
layout: post
title: "The Design and Implementation of the FreeBSD Operating System Second Edition Chapter 3.5"
date: 2014-11-23 22:47:06 +0900
comments: true
categories: book FreeBSD
---
Memory-Management Services

- プロセスは 3 つのセグメントで実行開始する
  - text
	- read-only
	  - 同一のファイルを実行する他のプロセスと共有
  - data
	- さらに 2 種類に分けられる
	  - 初期化済み
	  - 未初期化(bss)
	- 書き込み可
	- プロセスごとに private
  - stack
	- 書き込み可
	- プロセスごとに private
- FreeBSD は次のフォーマットの実行可能ファイルをサポートする
  1. interpreter が読むファイル
	 - 最初の 2 bytes が "#!" で interpreter の pathname が続く
	   - image activator が interpreter を走らせ、引数としてファイル名を渡す
  2. 直接実行可能なもの(AOUT, ELF, gzipped ELF)
	 - 実行可能ファイルは image activation(imgact) フレームワークで parse
	   - 登録された image activator のリストからマッチするものを見付ける
		 - 見付かったら対応する image activator が実行準備
- 直接実行可能なファイルのヘッダ
  - image activator が使う情報
	- ビルドしたアーキテクチャや OS の情報
	- statically or dynamically linked
  - text、初期化済み data、未初期化 dataのサイズ
  - デバッグ用の情報
- text、初期化済みデータがヘッダに続く
  - 未初期化データは必要に応じて 0 で fill されるだけなのでファイルには含まれない
- 実行開始
  - text をプロセス空間の下部に配置
	- virtual address space の 2 ページ目から
	  - 1 ページ目は null pointer access を失敗させるため invalid にしておく
  - 初期化済み data は text の次
  - その次に未初期化 data 用の領域
	- 0 filled
	- 未初期化 data の領域は sbrk システムコールで拡張可能
	  - 普通のユーザプログラムは使いやすい malloc() を使うけど
	- 元の data セグメントの上から拡張していく
	  - この領域を heap と呼ぶ
	  - PC では stack は top から bottom に向かって成長する
		- heap は逆
  - stack の次
	- 引数の数 (argc)
	- 引数の vector (argv)
	- process environment vector (envp)
	- 引数、environment の文字列自身
	- signal code
	  - システムがこのプロセスに signal を deliver した時に使う
	- 最後は ps_string
	  - ps がプロセスの argv を配置するのに使用
  - dynamic link している場合
	- dynamic loader が mmap で共有ライブラリ用の空間を割り当てる
	  - stack の administrative lower limit の下
- textと初期化済み data の全体をメモリにコピーすると実行開始までの latency が大きくなる
  - demand paging で回避
	- プログラムは必要に応じてページ単位で load
	  - ページはアドレス空間を等しい大きさに分割した小領域
	  - 最初にページにアクセスした時に page-fault trap 発生
		- page-fault handler がそのページを実行可能ファイルからメモリに読み込む
- プロセスは他にも global な resource を必要とする
  - スケジューリングや virtual memory 割り当ての情報
	- プロセスのアドレス空間は全て swap-out されるかもしれない
	  - メモリ上に戻すための情報が必要
	  - スケジューリングとかの情報は swap-out されていても maintainance しなくてはならない
  - descriptors 関連情報
  - page tables
	- 物理メモリの利用情報を記録
	  
	  
