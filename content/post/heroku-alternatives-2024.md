---
title: "無料で使える Heroku 代替サービス 2024版"
date: 2024-11-28T17:20:02+09:00
draft: false
categories: ["web"]
tags: [
  "heroku"
]
---

{{<figure src="/img/ss/heroku10.png" class="center" width="50%">}}

時は2024年、Heroku の無料枠が廃止されてから2年ほど経ちますが、その間にいろいろな代替サービス・プラットフォームが登場したり持ち上げられたりしては消えていきました。
無料として客を呼び込んだあと無償プランを廃止したサービスもいくつかあり、その中には救済措置として「レガシープラン」にコンバートした上で細々と無料提供を続けているサービスもあります (fly.io など)。しかし無料枠を続けているサービスはごっそり減ってしまいました。

月数ドル支払うだけで大変使い勝手のよい素晴らしい Heroku 代替サービスが多数あることも事実です。ぜひ数ドル払って快適な PaaS を使うことをお勧めします。が、無料というのは何事にも変え難い魅力があります。
というわけで、2024年ももうすぐ終わりになりそうですが「無料で使える Heroku 代替サービス 2024版」と題して無料で使えることを確認した Heroku 代替の PaaS (Platform as a Service) を3つほど紹介したいと思います (2025年になったらまた状況が変わってるかもしれませんしね...)。

なお、選定にあたって特定のフレームワークを要求するもの (Vercel など) や、無料枠では静的サイトホスティングしかできないもの (DigitalOcean App Platform など) は除外しています。

### 1. Render

{{<figure src="/img/ss/render1.png" class="center" width="50%">}}

ウェブサイト: https://render.com/

Heroku の無料枠が廃止された2022年当時から Heroku 代替の有力候補の一つとして (fly.io や Railway.app などと共に) 紹介されることの多かったサービスの一つです。
無料枠は「750時間/月」となっていますが、これは1サービスを 24/365 稼働させるのに十分な枠となっています (1ヶ月を31日としても 24h x 31 = 744h で収まる)。

デプロイ方法は Docker のほか、Express, Django, Ruby on Rails, Gin, rocket, Phoenix, Laravel とさまざまなフレームワークのデプロイテンプレートが用意されていて、GitHub や Docker Hub と連動した自動デプロイなども簡単に構成できます。

無料インスタンスは15分間アクセスがないと勝手に停止 (spinning down) してしまう点に注意する必要があります。一度停止すると起き上がるのに最大1分かかるため、事前に回避したいところです。どっかのサーバーから定期的にリクエストを投げるスクリプトを回したり、UptimeRobot や StatusCake など外形監視 SaaS を使って見守らせるのがおすすめです。
また、私はちょっと通信頻度高めなサービスを Render で運用していたのですが、Bandwidth リミットに達していないにも関わらず強制停止させられてしまうことが何度かあり、運用を諦めました。ヘビートラフィックなサービスは載せないほうが良いかもしれません。

周辺サービスとして、静的サイトのホスティング機能や Cron ジョブ機能も無料枠の範囲で提供されています。PostgreSQL の DBaaS もありますが、30日間のフリートライアルという位置付けなので注意が必要です。Redis も無料で使えますが、シングルインスタンス・永続化ストレージへのバックアップなしである点に注意してください。

利用にあたり、クレジットカードの登録は不要です。無料枠の限界に近づくとメールが届きます。
無料で使えるリージョンが豊富なのも嬉しいポイントです。執筆時点では US 3 リージョン (Oregon, Ohio, Virginia) のほかフランクフルトとシンガポールが使えます。
カスタムドメインも無料で使うことができます。

Heroku からのマイグレーションガイドが以下の場所にあります:

> Migrate from Heroku to Render – Render Docs
>
> https://render.com/docs/migrate-from-heroku

### 2. Koyeb

{{<figure src="/img/ss/koyeb1.png" class="center" width="50%">}}

ウェブサイト: https://www.koyeb.com/

2021年頃に登場した新し目の PaaS です。プランが細かく切られており、Eco プランのうち最小サイズのものが "Free Forever" として無料で使えます。
無料枠は CPU 割り当てが 0.1 と少なめですが、メモリは 512MB と標準的なサイズです。

デプロイ方法は GitHub リポジトリを連携して Dockerfile を探してビルドする方法のほか、Node.js, Python, Go, Ruby, PHP, Java, Scala を検出して構成する機能も備わっています。Docker Hub などのコンテナレジストリから取ってくることももちろんできます。

無料インスタンスは2分間アクセスがないと勝手にスリープしてしまうようですが、新しいトラフィックがきても 500ms で起き上がるそうなので、体感はするもののわざわざ外形監視するほどでもないように思います。おそらくプロセス状態を維持したままメモリからディスク等にオフロードしているのでしょう。

周辺サービスとして、0.25 vCPU, 1 GB RAM, 1 GB Disk の PostgreSQL サーバーを無料で1つ立てることができます。無料枠がある DBaaS としては枠がかなり大きめで嬉しいポイントに思います。

利用にあたり、クレジットカードの登録は不要です。
無料で使えるリージョンはワシントン DC とフランクフルトの2つです。
カスタムドメインも無料で使うことができます。

Heroku からのマイグレーションガイドが以下の場所にあります:

> Migrate from Heroku to Koyeb
>
> https://www.koyeb.com/tutorials/migrate-from-heroku

### 3. Back4app Web Development Platform

{{<figure src="/img/ss/back4app1.png" class="center" width="50%">}}

ウェブサイト: https://www.back4app.com/

Back4app は Firebase ライクな Backend as a Service を提供していることで知られていますが、Container as a Service も提供しています。
無料枠のメモリは 256MB とやや少なめですが、CPU 割り当ては 0.25 となっています。

デプロイ方法は Container Platform と言うだけあって Docker 一択です。Dockerfile がない場合は "dockerizing" をする必要があります。
といっても、たくさんのフレームワーク向けのガイドが用意されているので安心です。執筆時点で Node.js, Express, Python, Flask, Django, React, Next, Angular, Vue.js, Laravel, CakePHP, CodeIgniter, Symfony, Phoenix, Remix, Go, Deno, Ruby, Ruby on Rails, Java, Spring, C#, ASP.NET, Next.js, Meteor, RedwoodJS, Crystal, Rust のガイドがあります (これら全部知っている人はどれぐらいいるだろうか...)。

> How to create a Dockerfile - Back4app Containers
>
> https://www.back4app.com/docs-containers/how-to-create-a-dockerfile

無料インスタンスは30分間アクセスがないと勝手にスリープしてしまいます。起き上がるのにかかる時間は手元のアプリでは4秒ほどでしたが、ケースバイケースかもしれません。
気になる場合は、どっかのサーバーから定期的にリクエストを投げるスクリプトを回したり、UptimeRobot や StatusCake など外形監視 SaaS を使って見守らせるのがおすすめです。
ただし、無料枠の稼働時間が月600時間となっているので、適度にスリープする時間帯も混ぜておかないと月末にアクセスブロックを喰らってしまうおそれがある点に注意してください。

周辺サービスとして、看板サービスの Backend Platform には NoSQL データベースやファイルストレージ、認証基盤、通知基盤のほか関数実行基盤まで備わっていて、無料枠は 25000 リクエスト、250MB データストレージ、1GB 通信、1GB ファイルストレージとなっています。

利用にあたり、クレジットカードの登録は不要です。
無料で使えるリージョンは USA のみとなっています。
カスタムドメインも無料で使うことができます。

### まとめ

3サービスの比較を以下にまとめました。

| サービス | クレカ登録 | CPU | Mem | Bandwidth | リージョン | カスタムドメイン | スリープ |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Render | 不要 | 750時間/月 | 512MB | 100GB | Oregon, Ohio, Virginia, Frankfurt, Singapore | 対応 |あり (15分) |
| Koyeb | 不要 | 0.1 | 512MB | 100GB | DC, Frankfult | 対応 | あり (2分) |
| Back4app | 不要 | 0.25 | 256GB | 100GB | USA | 対応 | あり (30分) |

3サービス共通の特徴として、ダッシュボードはどのサービスも洗練されていてとても使いやすいです。ただし、どこも日本語には対応していません。

デプロイをプラットフォーム上で行う場合、大変時間がかかることが多いです。頻繁にデプロイしたい場合はフィードバックサイクルが遅くなるので、気になる場合はコンテナレジストリから最新の Docker イメージを自動で拾ってくるように構成した上でイメージを別の場所 (GitHub Actions など) でビルドしてアップロードしておくのがおすすめです。

その他未調査のサービスも挙げておきます (随時追加):

- [alwaysdata](https://www.alwaysdata.com/en/) - ディスク領域 100MB の仮想マシンを扱うような感覚で使えるサービス
- [Lade](https://www.lade.io/) - 無料枠のメモリが 128MB とだいぶ少なめ、DB も無料枠あり

こんなサービスもあるよというのがあればぜひ教えてください。この記事の2025年版の執筆につながるかもしれません。

Happy hacking!
