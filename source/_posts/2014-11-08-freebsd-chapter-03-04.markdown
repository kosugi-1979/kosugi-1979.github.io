---
layout: post
title: "The Design and Implementation of the FreeBSD Operating System Second Edition Chapter 3.4"
date: 2014-11-08 23:08:18 +0900
comments: true
categories: FreeBSD book
---

clock interrupts

- tick
	- 定期的な clock の割り込み
		- 時刻やタイマの更新をする
	- PC だと 1 秒に 1000 回
		- 本当に 1000 回やると無駄が多い
			- カーネルで action を取る必要がある tick を計算
				- そこに割り込みをスケジュール
	- tick の割り込みは hardware 割り込みの priority で
		- clock-device のプロセスにスイッチ後、hardclock() が呼ばれる
			- hardclock() にかける時間は短くしないといけない
				- あまり重要でないものは低 priority の software 割り込みハンドラ softclock() で行う
			- hardclock() でやること
				- 実行中のプロセスが virtual または profiling interval timer を持っているならタイマをデクリメントし、時間が来たら signal を発行
				- 前の hardclock() からの tick 数だけ現在時刻を更新
				- softclock() を走らせる必要があるなら softclock プロセスを実行可能にする
- Statistics and Process Scheduling
	- PC だと統計用の clock(statclock) を専用に持つ
		- システム clock が兼用するとそれに同期することで統計が不正確になるかも
		- 毎秒 127 ticks
			- tick 時に実行中のプロセスをチャージ
				- 4tick 分たまると priority 再計算
					- priority が下がったら再スケジューリング
		- tick の時点でシステムが何をしているのかの統計も集める
			- idle
			- ユーザモードで実行中
			- システムモードで実行中
		- 最後にシステム I/O の情報を集める
	- profiling 用の clock(profclock) もある
		- 1 つ以上のプロセスが profile 情報を要求したら走る
			- システム clock とは互いに素になる頻度
				- PC だと毎秒 8192 ticks
- Timeouts
	- hardclock() の完了時、softclock() でやることが残っていたら softclock が走るようにスケジュール
		- softclock() が準備する定期的なイベントには以下のものがある
			- real-time タイマの処理
			- drop したネットワークパケットの再送
			- モニタリングが必要な装置上の watchdog タイマ
			- システムのプロセス再スケジューリングイベント
		- CPU 利用に応じた各プロセスの CPU priority の上げ下げもやる
			- 1 秒に 1 回
	- 待っているイベントを表すのデータ構造は callout queue
		- 詳しくは本の Figure 3.2 を見て
		- プロセスがイベントをスケジュール
			- 呼ばれる関数の指定
			- その関数に引数として渡されるポインタ
			- イベントが起こるまでの clock tick 数
		- カーネルは queue の先頭部の配列を保持
			- それぞれの queue は特定の時刻を表現
			- 現在時刻の queue を指すポインタがある
				- その次の queue は 1 tick 先
					- 配列の最後まで行ったらまた最初から
						- t 要素の配列だったら同じ queue に t tick 先のもにも使用
			- hardclock() の度に callout queue の先頭ポインタを 1 増やす
				- queue が空でなければ softclock() が走るようにスケジュール
					- softclock() は現在の queue をスキャン
						- 各イベントが持っている時刻と現在時刻を比較
							- 一致したらイベントを削除して関数呼出し
