---
layout: post
title: "The Design and Implementation of the FreeBSD Operating System Second Edition Chapter 3.2"
date: 2014-10-11 11:48
comments: true
categories: book FreeBSD
---
- システムコールハンドラの仕事
	- パラメータのアドレスが正しいかの確認とカーネル空間へのコピー
	- システムコールを実装しているカーネルルーチンの呼出し
- Result Handling
	- 成功にしろ失敗にしろシステムコールは呼出し元に返る
		- 成功: carry bit 0
		- 失敗: carry bit それ以外
	- C の関数は返戻値を汎用レジスタにいれて返す
		- システムコールが失敗したら C のライブラリルーチンがシステムコールの返戻値を errno に格納
		- 返戻値用のレジスタに -1 を入れる
	- システムコールの失敗には 2 種類ある
		- カーネルがエラーを見付ける
		- 割り込み
			- イベント待ちで CPU を手放す
			- 途中でシグナルが届く
				- シグナルハンドラ初期化時に割り込まれたシステムコールの振舞いを指定
					- 再スタートするか
					- エラーを返すか
	- システムコールが割り込まれたら、シグナルがプロセスに配送される
		- システムコールをアボートする場合
			- エラーを返す
		- 再スタートする場合
			- 保存されているプログラムカウンタの値を trap のあった命令にセット
			  	- シグナルハンドラから返ってきたらシステムコールを再スタート
	- 再スタートには以下の前提がある
		- カーネルがプロセス空間のパラメータをいじっていないこと
		- 繰返し不可なことを行っていないこと
			- ターミナルからの読み込みとか
- Returning from a System Call
	- システムコール実行中、またはブロックされたシグナルで sleep 中に次のことが起こり得る
		- シグナルが post される
		- 他のプロセスがより高い priority を獲得
	- システムコール完了後、system-call exit code で上記のイベントが起こったかを確認する
	- sytem-call exit code
		- post されたシグナルをチェック
			- 次の場合はプロセスに post されない
				- デフォルトで無視するシグナル
				- 明示的に無視するよう設定
			- デフォルトアクションがあるならプロセス再開前に実行
			- シグナルを catch するようにしている場合
				- シグナルハンドラを実行
				- ハンドラの実行後にシステムコールから戻るところから再開
					- システムコール再スタートの時もある
		- 現在実行中のプロセスよりも高い priority のものがないか探す
			- あったら context switch する
				- 後で自分が実行再開される時にユーザプロセスに戻る
		- profiling している場合にはシステムコール内でかかった時間を計算
