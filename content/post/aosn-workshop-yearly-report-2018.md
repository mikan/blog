---
title: "AOSN読書会 年次活動報告 2018"
date: 2018-09-26T22:32:02+09:00
draft: false
categories: ["life"]
---

{{<figure src="/img/ss/aosn-report1.png" class="center" width="60%">}}

先日、私が世話人？を務めるオンライン読書会「[AOSN読書会](https://aosn.ws)」の合宿が北海道の登別温泉で開催されました。

合宿は毎年開催しており、温泉と地ビールのある場所を選ぶのが恒例となっています。
過去の開催地は2015年は有馬温泉 (兵庫県)、2016年は鬼怒川温泉 (栃木県)、2017年は湯涌温泉 (石川県) となっており、いずれも大盛り上がりとなりました。

世話人の私は毎年ここで1年間の活動データを Elasticsearch と Kibana を用いて可視化したものを用いて活動報告としてまとめています。以前公開した [AWS Lambda + Chart.js の仕組み](/2018/01/23/data-visualization-on-static-sites-with-aws-lambda-go-and-chart-js/)は課題本あたりのデータ可視化を行っていますが、課題本をまたいだ全期間や年単位などの絞り込みにはまだ Kibana は手放せません。

それでは振り返りに参りましょう。

AOSN読書会ではこの1年間で以下の本を読み終えました。なお、AOSN読書会では輪読会と討論会の2部構成でそれぞれ別の本を読みます。輪読会部をAパート、討論会部をBパートと呼んでいます。Bパートは予習前提です。

{{<figure src="/img/ss/aosn-report3.png" class="center" width="60%">}}

「[プログラミング Elixir](https://amzn.to/2QVCjsi)」は、それほど厚い本ではないにも関わらず、読了までに読書会最長の全44回 (つまり44週) を要しました。
本読書会では時間内で本文を通読するだけでなく、サンプルコードや練習問題などを実際に写経したり解いたりし、その結果をレビューしあったりすることがあります。
この Elixir 本は練習問題が豊富なため、とても多くの時間がかかりました。
でもそのおかげで、Elixir が持つ使い勝手のよい関数型言語の魅力をダイレクトに感じることができ、参加者の間でも満足度は高かった本ではないかと思います。
2016年2月以来の最大同時参加者数8名も1度記録しました。

「[Real World HTTP](https://amzn.to/2QUSvdv)」のほうは、扱っているテーマや内容は悪くない本ですが、途中で読むのを止める人も出てきたりして、新技術のキャッチアップを第一の目標とする本読書会としては知的好奇心を維持し続けるのが難しい本であったように思います。
また本書はエラッタや技術的間違いと思われる箇所も比較的多かったため、実際に出版社に報告も行い正誤表にも貢献しました。しかし技術的間違いの指摘に関しては著者から反応はありませんでした。

「[現場で役立つシステム設計の法則](https://amzn.to/2R1zVQJ)」は討論会であるBパートで扱いましたが、提案者の狙い通りの展開となりました。
というのも、発売直後から賛否両論が激しいとの前評判があったため、本をダシに意見を交換し合うBパートにはピッタリだったからです。
とはいえ、エンジニアリング的には初歩的な内容が多いためディスカッションのレベルも上がらず、次からはこうした本は避けたほうが良いのではとも思います。

{{<figure src="/img/ss/aosn-report4.png" class="center" width="90%">}}

ここからは Kibana のスクショを用いて活動を振り返ります。
まずは Y-Axis を参加者数・非参加者数のユニークカウントの合計のパーセンテージ、X-Axis を日にした参加率のバーチャート (全期間) です。
白いエリアは読書会の開催が何らかの要因でスキップされていることを示しています。
最近は途中離脱などで参加者不足による流会が頻発しており、運営上の課題になっています。

{{<figure src="/img/ss/aosn-report5.png" class="center" width="90%">}}

次はAパートの参加者推移 (全期間) です。
Y-Axis が参加者数のユニークカウントです。課題本ごとにシリーズ分割をしており、色が変わっているところが別の課題本に変わったことを示しています。以前より新しい課題本になると参加率が回復する傾向があります。
去年はついに二人開催 (サシ読み) を解禁し、本の残りを強引に読み切って次回本に行くといった運営判断をすることがありました。

{{<figure src="/img/ss/aosn-report6.png" class="center" width="90%">}}

こちらは先程のグラフのBパート版 (全期間) です。
本読書会はAパートのみ参加やBパートのみ参加もできるのですが、B参加者は固定客が多くあまり変動がありません。
なおBパートはサシではやりづらさが増すため、最少催行人数3人をなるべく守って (満たない場合は流して) います。

{{<figure src="/img/ss/aosn-report7.png" class="center" width="90%">}}

最後のバーチャートは人別の参加回数ランキング (直近1年間) です。
去年の報告のときには常連とそれ以外の差はさほどなかったのですが、今年になって広がってしまいました。
課題本要因だったり、外的 (参加者のプライベート等の) 要因だったりいろいろ考えられますが、いずれにせよ良い状態ではありません。
継続的に参加できるようにできることをみんなで考えていかねばと思っています。

実際の資料は以下を御覧ください (Go の `present` を利用)。

> AOSN読書会 年次活動報告 2018
>
> https://go-talks.appspot.com/github.com/mikan/talks/aosn-report-2018.slide

AOSN読書会では参加者を募集しています。
現在の課題本はAパートが「[プログラミングRust](https://amzn.to/2QZH8B1)」、Bパートが「[ベタープログラマ](https://amzn.to/2N3YMA9)」です。
Rust ははじまったばかりなので、参加もしやすい時期かと思います。
参加方法や Q&A を以下に掲載していますので、ご興味のある方は是非確認・エントリーしてみてください。

> 🔰エントリー - AOSN読書会
>
> https://aosn.ws/3-entry/

Happy hacking!