---
layout: post
title: "The Design and Implementation of the FreeBSD Operating System Second Edition Chapter 3.6"
date: 2014-12-13 13:22:13 +0900
comments: true
categories: book freebsd
---
Timing Services

- 実時間
    - Epoch として知られる 1970 年 1 月 1 日(UTC)からのオフセットを gettimeofday システムコールで取得
    - 最近の CPU はバッテリーバックアップ式の time-of-day レジスタを持ってる
        - プロセッサが off にされても走り続ける
            - システムの boot 時に現在時を調べるのに使用
                - 以降は clock 割り込みで維持
                    - tick 分の microsecond を global time variable に加える
- 外部表現
    - システムの外側には常に microsecond
        - resolution independent
        - 内部では自由な tick rate をとれる
            - clock-interrupt handle と resolution の tradeoff
    - ファイルシステムのタイムスタンプは Epoch からの UTC offset
        - local time への変換はシステムの外側で行う
- 時刻の調整
    - ネットワーク上の全マシンで同一の時刻を保持しているのが望ましい
        - プロセッサの clock よりも正確な時刻にもできる
    - 異なるマシン上のプロセスはプロセッサの時刻をネットワーク上の時刻に合わせたい
        - settimeofday システムコール
            - 時刻が逆戻りするかも
                - make とかにとっては都合が悪い
        - adjtime システムコール
            - time delta を取る(正負あり)
                - 時刻が合うまで時間の進みを 10 % 早めたり遅らせたりする
- Interval Time
    - システムは 3 つのインターバルタイマを提供する
        - real timer
            - タイマが切れると SIGALRM シグナルをプロセスに配送
            - softclock() でメンテナンスされている timeout queue から走る
        - profiling timer
            - 実行の統計プロファイル用
            - タイマが切れると SIGPROF シグナルをプロセスに配送
            - profclock() の度に現在実行中のプロセスが使っているかチェックして、0 になったらシグナル送信
                - ユーザモード、カーネル内の双方でタイマをデクリメント
        - virtual timer
            - ユーザモードでの実行中にのみ走る
            - タイマが切れると SIGVTALRM シグナルをプロセスに配送
            - profclock() 内でも実装されている
                - 現在のプロセスがユーザモードで実行中の場合にのみタイマをデクリメント
