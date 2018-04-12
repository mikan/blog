---
title: "PlantUML 埋め込み AsciiDoc の Gradle を用いた HTML 一括変換"
date: 2018-02-23T22:16:01+09:00
draft: false
categories: ["tools"]
tags: [
  "gradle",
  "asciidoc",
  "plantuml",
  "jruby",
  ]
---

最近仕事で機能仕様書や設定手順書などの開発ドキュメントの AsciiDoc 化を推し進めています。
私個人は今まで Markdown や reStructuredText (reST) を用いた設計文書の記述をよくやっていたのですが、たまたま最近 ([JJUG CCC 2017 Fall](/2017/11/19/jjug-ccc-2017-fall/) や社内のディスカッションなどで) AsciiDoc を推す声を多く聞いたこともありこっちもやってやるかと思うようになりました。

{{< figure src="/img/logo/asciidoc.png" title="AsciiDoc" class="center" link="http://asciidoc.org/">}}

また、併せて PlantUML も導入されました。テキストで UML を表現できるため Git フレンドリーなのと、そこそこ綺麗にレンダリングでき、さらにレイアウトがツール任せなので個人差が出にくいのがかえってチーム開発では嬉しかったりします。

もちろん、複雑な UML を書くときは愛用 (?) の [astah professional](http://astah.change-vision.com/ja/product/astah-professional.html) で書くこともありますが、最新の macOS での操作性が不具合対応のため劣化したのと、以前からマルチディスプレイ環境でフリーズすることが多く、最近はよほどのことがないかぎり起動しなくなりました。もうライセンスを更新することはないかなぁ (Change Vision さんごめんなさい・・・！)。

{{< figure src="/img/logo/plantuml.png" title="PlantUML" class="center" link="http://plantuml.com/">}}

AsciiDoc はプロセッサとして Ruby 製の [Asciidocor](https://asciidoctor.org/) がポピュラーで、さらにこの拡張である asciidoctor-diagram を導入することで AsciiDoc 内に PlantUML レンダー結果を埋め込むことができます。
これは素晴らしいですね。しかし AsciiDoc/Asciidoctor 自体はディレクトリ内のファイルを一括で (しかも再帰的に) HTML 化することはできません。そこで便利なのが・・・

{{< figure src="/img/logo/gradle.png" title="Gradle" class="center" link="https://gradle.org/" width="50%">}}

Gradle です。

Asciidoctor には Gradle Plugin があり、名前はずばり Asciidoctor Gradle Plugin です。

> Asciidoctor Gradle Plugin | Asciidoctor
>
> https://asciidoctor.org/docs/asciidoctor-gradle-plugin/

Maven Plugin もあるようですね。Maven にこだわりたいというお方はここでお別れです:grin:

Asciidoctor Gradle Plugin は、Ruby 製の Asciidoctor を JRuby で動かしてしまうというものです。Gradle (Groovy の DSL) から JRuby が呼ばれて Ruby が動くのです。夢のような技術ですね:sparkles:
なお、JRuby を用いた Asciidoctor の Java バインディングを AsciidoctorJ と呼びます。

### Asciidoctor 単体

それでは、まずは AsciiDoc 単体で動作確認してみましょう。

`build.gradle` を作ります。

```groovy
apply plugin: 'org.asciidoctor.convert'
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'org.asciidoctor:asciidoctor-gradle-plugin:1.5.3'
    }
}
```

デフォルトの対象ディレクトリは `src/docs/asciidoc` です。それに習ってファイルを配置してみます。

```
$ tree
adoc-test
├── build.gradle
└── src
    └── docs
        └── asciidoc
            ├── func.adoc
            └── index.adoc

4 directories, 3 files
```

:bulb: ソースディレクトリを変更したい場合は、 `build.gradle` に `asciidoctor{ sourceDir = file('./docs') }` のようにパスを教えてあげる必要があります。

adoc の中身はなんでも良いですが、今回はサンプルとしてこんなものを置いてみました (`index.adoc` の中身)。

```adoc
= Asciidoc てすと
mikan <mikan@aosn.ws>
v0.1, 2018-02
ifdef::env-github,env-browser[:outfilesuffix: .adoc]
:toc:

== 仕様書

* link:func{outfilesuffix}[機能仕様書]

== 用語集

.一般用語
[format="csv",options="header"]
|===
用語,読み方,説明
Asciidoc,あすきーどっく,文書マークアップ言語のひとつ
PlantUML,ぷらんとゆーえむえる,テキストベースの UML 記述ツール
|===
```

:bulb: HTML 出力した際のリンクを `.adoc` ではなく `.html` とするために `outfilesuffix` 変数の切替えを利用しています。上記のようにすることで、GitHub や IDE 等におけるレンダリングは `.adoc` に、HTML 出力結果は `.html` となります <sup>[参考文献1]</sup>。

準備ができたら、ビルドしてみましょう。

```
$ gradle asciidoctor
```

`BUILD SUCCESSFUL` と表示され、`build/asciidoc/html5` に html ファイルができていれば成功です。早速開いてみましょう。

{{< figure src="/img/ss/asciidoctor1.png" title="Asciidoctor 出力 HTML" class="center" width="80%">}}

ばっちりですね:sparkles: 文書間のリンクもちゃんと `.html` になっていることが確認できるはずです。

:bulb: アウトプットディレクトリを変更したい場合は、 `build.gradle` に `asciidoctor{ sourceDir = file('./docs') }` のようにパスを教えてあげる必要があります。

さて、これで仕事が終わるなら簡単すぎてブログ記事になりません。**問題はここから**なのです・・・！

### PlantUML を埋め込んでレンダリングする

先に PlantUML をセットアップしておきます。今回レンダリングするために PlantUML と Graphviz のセットを導入しておく必要があります。Mac の Homebrew であれば `brew install plantuml` と叩くだけで準備は完了です。Windows は [Chocolatey](https://chocolatey.org/) であれば両方パッケージが揃っていて手軽です。既に IDE 上でプレビューするためのプラグイン等の環境構築を行っていれば、おそらくそのまま Gradle からも呼び出せることでしょう (私のおすすめは IntelliJ の AsciiDoc プラグインです)。

さて、先ほどのサンプルでは `index.adoc` ともう一つ `func.adoc` というファイルを用意していました。そちらに PlantUML を書き込んでみます (中身にツッコミはなしでオナシャス)。

```adoc
= 機能仕様書
mikan <mikan@aosn.ws>
v0.1, 2018-02
:toc:

== ユースケース図

[plantuml]
----
@startuml
:ユーザー: -> (本を借りる)
:ユーザー: -> (本を返却する)
@enduml
----
```

そして、これをビルドする `build.gradle` ですが... 最終的に使うのは冒頭で軽く触れたように asciidoctor の diagram 拡張になります。

> Asciidoctor Diagram | Asciidoctor
>
> https://asciidoctor.org/docs/asciidoctor-diagram/

ですが、これ自体はそもそも Ruby で require して使うものです。JRuby だとどうなる!? さらにそれを Gradle から指示するには!?

今回役にたったのは JRuby Gradle Plugin です。Asciidoctor Gradle Plugin との合わせ技が可能でした。

> JRuby/Gradle
>
> http://jruby-gradle.org/

・・・そして試行錯誤を重ねて私の手元で動くところまで書き加えた `build.gradle` はこんな感じになりました。

```groovy
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.github.jruby-gradle:jruby-gradle-plugin:1.5.0'
        classpath 'org.asciidoctor:asciidoctor-gradle-plugin:1.5.3'
        classpath 'org.asciidoctor:asciidoctorj-diagram:1.5.4.1'
    }   
}

apply plugin: 'org.asciidoctor.convert'
apply plugin: 'com.github.jruby-gradle.base'

dependencies {
    gems 'rubygems:asciidoctor-diagram:1.5.7'
}

asciidoctor {
    requires = ['asciidoctor-diagram']
}
```

それでは gradle を叩いてみましょう！

```
$ gradle clean asciidoctor
```

{{< figure src="/img/ss/asciidoctor2.png" title="Asciidoctor + PlantUML 出力 HTML" class="center" width="80%">}}

レンダリング結果が埋め込まれれば成功です！:confetti_ball:

それにしても、Groovy に JRuby に Java に・・・すごい合わせ技でした (※ PlantUML は Java でできている)。

### トラブルシューティング

喜びはつかの間、このソースを GitHub に置いて push 通知を Webhooks で飛ばし適当なサーバーで Gradle ビルドを実行、生成 HTML をその場でホスティングするという仕組みを社内で構築、展開したのですが、そのときにいくつかのトラブルに遭遇しました。手元で実行するまでの間にハマったものもふくめ、トラブルシューティングとして記録を残しておきたいと思います。

#### Java のバージョン

これまで紹介した `build.gradle` では **Java 8** でしか動作せず、Java 7 環境や 9 環境が設定されている環境では動きませんでした。つらい！！まあ、JRuby というだけで嫌な予感がしてましたけどね・・・。複数バージョン入っている場合は `JAVA_HOME` をいじりながらやりくりする必要があります。

#### Asciidoctor Gradle Plugin のバージョン

asciidoctor-gradle-plugin は **1.5.3** をお使いください！これより新しくすると (1.5.7 等)、PlantUML レンダリングが最初の一発だけしか正しく行われないという現象に遭遇することがあります <sup>[参考文献2]</sup>。

#### PDF 埋め込み機能

Asciidoctor には PDF を出力する機能を追加することができますが、今の所 PlantUML のレンダリング結果を拾ってくれません (`'org.asciidoctor:asciidoctorj-pdf:1.5.0-alpha.16'` で検証)。
ログを見ると png レンダリングされたファイルがある `outputDir` のパスではなく `sourceDir` のほうを読みに行ってしまっている様子。 ~~あと一歩な気がする！~~
そこで、 `asciidoctor` ブロックを以下のように設定し、正しいパスを教えてあげます。

```groovy
asciidoctor {
    backends = ['html5','pdf']
    requires = ['asciidoctor-diagram']
    attributes "imagesdir": buildDir
}
```

`buildscript` の `dependencies` に次を追記するのもお忘れなく。

```groovy
classpath 'org.asciidoctor:asciidoctorj-pdf:1.5.0-alpha.16'
```

:sparkles: _2018/04/13 加筆しました (Thanks [@osamus](#comment-3851233493))_

#### org.jruby.ext.openssl がないと言われる

やっつけですが、`build.gradle` の `buildscript` の `dependencies` に以下を追加することで回避できたことがあります:

```groovy
classpath 'rubygems:jruby-openssl:0.9.21'
```

#### PlantUML レンダリング結果で日本語が豆腐になる

Java が日本語フォントを認識していないせいです。JRE の `/jre/lib/fonts` に `fallback` というディレクトリを作り、そこにフォントのシンボリックリンクを突っ込むことで解決できます <sup>[参考文献3]</sup>。

エンジニアフレンドリーなドキュメンテーション環境の追求は続く...

Stay tuned & Happy hacking!

#### 参考文献

1. [Frequently Asked Questions (FAQs) and Troubleshooting](http://asciidoctor.org/docs/faq/) - asciidoctor.org
2. [Extentions dissapear with gradle daemon after first run · Issue #222 · asciidoctor/asciidoctor-gradle-plugin](https://github.com/asciidoctor/asciidoctor-gradle-plugin/issues/222)
3. [Java 実行環境のフォント - ArchWiki](https://wiki.archlinux.jp/index.php/Java_%E5%AE%9F%E8%A1%8C%E7%92%B0%E5%A2%83%E3%81%AE%E3%83%95%E3%82%A9%E3%83%B3%E3%83%88)
4. [asciidoctor/asciidoctorj-pdf: AsciidoctorJ PDF bundles the Asciidoctor PDF RubyGem (asciidoctor-pdf) so it can be loaded into the JVM using JRuby.](https://github.com/asciidoctor/asciidoctorj-pdf)
