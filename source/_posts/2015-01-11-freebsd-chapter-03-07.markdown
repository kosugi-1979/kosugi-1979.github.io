---
layout: post
title: "The Design and Implementation of the FreeBSD Operating System Second Edition Chapter 3.7"
date: 2015-01-11 13:32:02 +0900
comments: true
categories: book freebsd
---

Resource Services

- Process Priorities
    - FreeBSD は default で share scheduler でスケジューリング
        - 最近 CPU を使っていないプロセスが優先
            - 短い時間しか実行していないプロセスを好む傾向
                - インタラクティブなやつとか
        - カーネル内部でプロセスの priority 管理
            - プロセス毎に持つ nice 値が優先度計算に影響する
                - CPU のシェアを低くしたいなら正
                - CPU のシェアを高くしたいなら負
            - nice 以外も多くの要素が priority に影響する
                - 最近プロセスが使った CPU 時間
                - 最近プロセスが使ったメモリ量
                - 現在のシステムの負荷とか
    - real-time scheduler も使用可能
- Resource Utilization
    - カーネルはプロセスが使用しているリソースの統計を収集
        - 親プロセスは wait システムコール類経由でその統計を利用できる
        - プロセスが使用しているリソースは getrusage システムコールで取得
            - プロセスが使ったユーザ、システム時間
            - プロセスのメモリ使用量
            - プロセスのページング/disk I/O アクティビティ
            - コンテキストスイッチの回数(自主的なものとそうでないもの)
            - プロセス間通信の量
        - カーネルの全体でリソース使用情報を収集
            - CPU 使用量は statclock()
            - プロセスの priority 再計算時にメモリ使用量をサンプリング
            - vm_falut() はページングのアクティビティを再計算
            - I/O のアクティビティは転送開始の度に収集
            - プロセス間通信のアクティビティは送受信時に更新
- Resource Limits
    - カーネルはプロセス毎のリソースを制限できる
    - リソースには soft limit と hard limit という 2 種類の制限
        - soft limt
            - 全ユーザが変更可能(0 から hard limit まで)
            - 超えると signal が配送される
                - 普通は terminate するけど、プロセスの側で catch なり ignore もできる
                    - ignore した上で確保済みのリソース解放に失敗すると、これ以上のリソース要求はエラーになる
        - hard limit
            - 低くする方に変更可能
                - superuser のみ上げられる
    - リソース制限は一般的にコンテキストスイッチの近くで実行される
        - CPU time の制限はコンテキストスイッチ関数の中
        - スタックとデータセグメントは制限に達したら allocation が失敗
        - ファイルサイズの制限はファイルシステムが実行
- Filesystem Quotas
    - 個々のファイルサイズ制限の他にユーザやグループが使える総量に対して制限もかけられる
