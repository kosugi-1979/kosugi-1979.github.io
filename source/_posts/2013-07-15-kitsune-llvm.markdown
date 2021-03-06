---
layout: post
title: "きつねさんでもわかるLLVM"
date: 2013-07-15 21:06
comments: true
categories: [LLVM, book]
---
Tutorial を読んで Kaleidoscope のコードを入力したくらいの事前知識で読んだ。達人出版会の版は未読のため、違いは分からず。  

 まずは全体的に気になった点から。「〜思われます」、「〜ようです」といった曖昧な表現が目についた。筆者も細かいことまで調べきれていないのであろうが、技術書でこのような表現に出会うと少し不安になるので、できれば避けて欲しかった。

 内容に関しては、上記のように Tutorial を読んだ身では 1-5 章は不要だった。というより 5 章の出来が残念。この章では DummyC という小さな言語処理系を作成するのだけれど、言語に持たせる機能の取捨選択に関して疑問がある。このあたり、Kaleidoscope と比べると分かりやすいかもしれないので、以下に比較用の表を載せる。

.| DummyC  | Kaleidoscope 
-----:|:---------:|:-------------:
 型  | int のみ | double のみ
 条件分岐   | なし        | あり 
 ループ     | なし        | あり
 演算子定義 | なし        | 単項、2項
 関数       | あり        | あり
 変数       | あり        | あり 
 Parser     | 再帰降下    | 再帰降下 + 演算子順位


上記のうちで特に問題なのは DummyC 側に条件分岐とループが存在していないこと。そのため、DummyC では制御構造の記述がまったくできず、実行は常に straight-forward なものになり、toy 言語としてのサンプルプログラムすらまともに作れない。実際、本書の中にある DummyC のプログラムは数字を出力する程度(ちなみに Kaleidoscope はマンデルブロ集合の描画を行う)。それにもまして分岐しないということは当然合流もしないので、せっかく 4 章で SSA 形式だの phi 関数だのと名前を出したり、条件文やループがどのような LLVM IR になるのかを説明したにもかかわらず、続く 5 章で作ってみた DummyC 処理系ではそれが全く登場しないことになってしまっている。  

こうした印象は 6 章に入っても続く。6 章では Pass を作りながら LLVM IR の解析を行う内容になっている。前述のように DummyC は一切の制御構造を持たず、6 章で扱うループ回数を数えるという Pass の作成には不適なようで、解析対象となる LLVM IR は clang を使って C で書いたプログラムから作成する。しかも、ここでしばらく御無沙汰になっていた phi 関数が現れるので、5 章に真面目に取り組んだ人ほど唐突な印象を受けるかもしれない。本書の中で最も紙数を費やしているのが 5 章であるだけにその結果が中で閉じてしまっているのはやはり残念。  
結果として各章の独立性は高まったかもしれないが、その分一貫したストーリーは失われている。便覧ならともかく、ステップアップで学ぶ入門書においてそのような構成にするのはマイナスの効果の方が大きいように思われる。  
できれば DummyC で出力した LLVM IR に対して 6 章の Pass を適用する構成にして欲しかった。このあたりは Kaleidoscope の Tutorial は良くできている。英語は死んでも読みたくないという人でなければ、Tutorial の方をやるべき。ただしあれはあれで少し古いため Header file の場所が変わっていて #include を書き変える必要があったり、コンパイルオプションに "-rdynamic" を渡さないと外部関数の呼出しに失敗するなど問題もあるけれど。

ちなみに p.56 で Kaleidoscope を

> KaleidoScope という簡単な関数型言語をターゲットにした

と紹介しているが、first class な関数もなく、サイトでも

> Kaleidoscope is a procedural language

とちゃんと手続き型と明示しているので勘違いしないように。変数を導入するのが Tutorial の後半だから筆者はそう思ったのかも。

6 章そのものに関しては前述の 5 章とのつながりが感じられない点を除けば分かりやすい。"[Writing an LLVM Pass](http://llvm.org/docs/WritingAnLLVMPass.html)" を読んでいないので、これを読んだ後ではまた受ける印象も変わるかもしれないが、Pass の作り方に関してよくまとまっているように思う。

7 章は実のところ入門書に徹するのであれば正直ない方が良いのだけれど、本書の中で一番面白い部分であるのも確か。と同時にすぐには役立たせにくい章でもある。以前の章との係わりも薄い。ここでも branch は使っていないので、もしかしたら当初は DummyC からここまで一本芯を通した構成にするつもりだったのかと思ってしまうのは勘繰り過ぎか。  
個人的にこの章に書かれている情報だけでは説明が簡素過ぎてバックエンドを書ける気はせず、とりあえず雰囲気をつかむのが精一杯といった感じなので、これだけ別冊にしてもっと詳しく書いてくれても良いのではないかとも思う。やっていることはかなりシンプルではあるが MIPS ライクな CPU を想定してそれ向けの Native Code を出力するバックエンドを作成するというもので、いきなり内容がディープになる。何らかのアセンブリ言語に一度も触れたことがない人には雰囲気をつかむことすらきびしいかもしれない。

最後に割とどうでも良いことを 2 つほど書いておく。  
表紙を見る限り一昔前に流行ったいわゆる萌え本に思えるが、中身は普通の技術書で、絵は単なる賑やかし。どうせなら、説明が一段落ついた所に絵を入れるなりして「ここで一息つけますよ」という目印にするくらいはして欲しかった。理想は名著『CPU の創り方』のように挿絵と内容がマッチしていることだけれど、あれは文章と絵を同じ人が担当しているからできることであって、そこまで求めるのは流石に酷か。ところで、表紙の絵、特に黄色い方の足が長すぎる気がするのですが、気のせいでしょうか。  
もうひとつ、「ミドルエンド」って言葉は一般的な用語として定着しているの? GCC Wiki なる所には MiddleEnd があるけれど、ドラゴンブック含め手元のコンパイラ本全部の索引を見てもそんな言葉はないのだけれど。端でもないのにエンドって。入門書では広く浸透している用語以外は避けるべきと考えているのでどうにも気になった。
