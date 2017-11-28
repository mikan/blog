---
title: "PMD Apex / PMD Visualforce で Salesforce のコードを静的解析する"
date: 2017-11-28T21:42:48+09:00
draft: false
categories: ["salesforce", "tools"]
tags: [
  "force.com",
  "apex",
  "visualforce",
  "gitlab",
  ]
---

Salesforce のカスタマイズで用いる独自言語、Apex と Visualforce のコードを静的解析できるツール PMD Apex / PMD Visualforce を使ってみたので、ハマりどころや CI の工夫などをまとめてみます。

{{< figure src="/img/logo/pmd.png" class="center" width="40%" link="https://pmd.github.io/" title="PMD">}}

PMD といえば、Java 界隈では言わずと知れた静的解析ツール。同じく Java 界隈でよく使われる [FindBugs](http://findbugs.sourceforge.net/) はソースコード (`*.java`) ではなくコンパイルされたクラスファイル (`*.class`) を解析対象とするため他の言語に移植が困難ですが、対して PMD はソースコードを AST (抽象構文木) 化して解析するため様々な言語に適用できるツールキットになるという特徴があります。
PMD が Apex/Visualforce に対応しているのを同僚から教えてもらったときは「え、対応してたんだ！」とびっくりしたものですが、思えば Apex はコンパイル＆リンクを Force.com サーバーで行うため、クライアントサイドで解析するには PMD は大変良いアプローチなわけですね。

蛇足ですが、PMD ロゴに添えられている「**Don't shoot the messenger**」とは英語圏のことわざで「悪い知らせを伝えてくる人に怒ってはいけない」といった意味のようです。何が言いたいかは・・・わかりますよね:smile:

> don't shoot the messenger Meaning in the Cambridge English Dictionary
>
> https://dictionary.cambridge.org/dictionary/english/don-t-shoot-the-messenger

さて、本記事は事前条件として Apex/Visualforce ソースコードが手元にある場面を想定しています。以下のようなツールが役立つでしょう。Salesforce DX はとってもエンジニアフレンドリーですし積極採用していきたいところですね:sparkles:

- [Salesforce DX](https://developer.salesforce.com/docs/atlas.ja-jp.sfdx_setup.meta/sfdx_setup/sfdx_setup_intro.htm)
- [Force.com Migration Tool](https://developer.salesforce.com/docs/atlas.ja-jp.daas.meta/daas/meta_development.htm)
- [Force.com IDE](https://developer.salesforce.com/page/Force.com_IDE)

それでは PMD をセットアップしてみたいと思います。素晴らしいことに PMD のディストリビューションには最初から Apex も Visualforce も備わっています。従って、あなたがすべきことは PMD をダウンロードし、ソースコードの在り処、期待する出力フォーマット、そして適用したいルールセットを添えて実行するだけです。

```bash
wget https://github.com/pmd/pmd/releases/download/pmd_releases%2F5.8.1/pmd-bin-5.8.1.zip
unzip pmd-bin-5.8.1.zip
alias pmd="$HOME/pmd-bin-5.8.1/bin/run.sh pmd"
pmd -d /path/to/src -f html -R apex-apexunit,apex-braces,apex-complexity,apex-performance,apex-security,apex-style,vf-security
```

備考:

- `-d` はいろんなメタデータのディレクトリがある上のディレクトリ (つまり `src`) を指定すれば OK です。trigger だの component だのは指定せずとも拾ってくれます。明示する場合は `-d` オプションの代わりに `-fileset` オプションでパターンをカンマ区切りで指定します<sup>[参考文献1]</sup>
- `-l` で言語を指定することができますが<sup>[参考文献1]</sup>、この例ではあってもなくても結果が変わりませんでした
- `-f` は出力フォーマットを指定する項目で、以下の選択肢があります<sup>[参考文献1]</sup>:
  - `codeclimate`, `csv`, `emacs`, `html`, `ideaj`, `summaryhtml`, `text`, `textcolor`, `textpad`, `vbhtml`, `xml`, `xslt` (`xsltFilename` プロパティも指定), `yahtml` (`outputDir` プロパティも指定)

`-f html` で HTML レポートを出力すると、こんな感じの質素な HTML が**標準出力**されます。

{{< figure src="/img/ss/pmd-report1.png" class="center" width="80%" title="PMD HTML Report">}}

`-f vbhtml` で HTML レポートを出力すると、こんな感じの HTML が標準出力されます。ファイルごとにまとまった感じで出力されていいですね。ちなみに vb とは Vladimir Bossicard (開発者の名前) のようです。

{{< figure src="/img/ss/pmd-report2.png" class="center" title="PMD HTML Report (VB)">}}

`-f yahtml` は Yet Another HTML というモードで、ディレクトリに複数の HTML 群を吐いてくれるモードですが、残念ながら Apex/Visualforce ではいい感じの HTML が生成されませんでした。なお、出力ディレクトリは `-P outputDir=xxx` で場所を教える必要があります。さらに、指定したディレクトリは既に存在する必要があります (通常は直前に `mkdir` する)。

`-f xml` は Jenkins のプラグインで集計する際などに使うフォーマットです。

[Visual Studio Code](https://www.microsoft.com/ja-jp/dev/products/code-vs.aspx) をお使いの方は、開いたり保存したりしたときに PMD Apex を自動的に掛ける素晴らしいプラグインがあります:

> ChuckJonas/vscode-apex-pmd: PMD static analysis for Apex in vscode
> 
> https://github.com/ChuckJonas/vscode-apex-pmd

さて、ここまで書いたところで今更なのですが、私の手元の環境では重大な問題が発生していました。なんと、Java SE 9 では Apex 解析部分が正常に動作しないようです (PMD Java とかは問題なし)。以前出たバグの再発っぽいですが、詳細は解析中。今はひとまず Java SE 8 / OpenJDK 8 で回避しましょう。上述の VSCode プラグインも Java 9 環境では全く吠えてくれません:sweat_smile:

```
Exception in thread "main" java.lang.IncompatibleClassChangeError: Inconsistent constant pool data in classfile for class apex/jorje/semantic/symbol/member/method/MethodTable. Method lambda$static$64(Lapex/jorje/semantic/symbol/member/method/MethodInfo;)Z at index 85 is CONSTANT_MethodRef and should be CONSTANT_InterfaceMethodRef
	at apex.jorje.semantic.symbol.member.method.MethodTable.<clinit>(MethodTable.java:38)
	at apex.jorje.semantic.symbol.type.ModifierTypeInfo$Builder.build(ModifierTypeInfo.java:119)
	at apex.jorje.semantic.symbol.type.ModifierTypeInfos.<clinit>(ModifierTypeInfos.java:54)
	at apex.jorje.semantic.ast.modifier.ModifierGroups.<clinit>(ModifierGroups.java:27)
	at apex.jorje.semantic.symbol.type.TypeInfos.<clinit>(TypeInfos.java:40)
	at apex.jorje.semantic.symbol.member.variable.TriggerVariableMap.<clinit>(TriggerVariableMap.java:46)
	at apex.jorje.semantic.symbol.resolver.StandardSymbolResolver.<init>(StandardSymbolResolver.java:80)
	at apex.jorje.semantic.compiler.CompilerContext.<init>(CompilerContext.java:49)
	at apex.jorje.semantic.compiler.ApexCompiler.<init>(ApexCompiler.java:57)
	at apex.jorje.semantic.compiler.ApexCompiler.<init>(ApexCompiler.java:37)
	at apex.jorje.semantic.compiler.ApexCompiler$Builder.build(ApexCompiler.java:210)
	at net.sourceforge.pmd.lang.apex.ast.CompilerService.compile(CompilerService.java:95)
	at net.sourceforge.pmd.lang.apex.ast.CompilerService.visitAstsFromStrings(CompilerService.java:90)
	at net.sourceforge.pmd.lang.apex.ast.CompilerService.visitAstFromString(CompilerService.java:78)
	at net.sourceforge.pmd.lang.apex.ast.ApexParser.parseApex(ApexParser.java:42)
	at net.sourceforge.pmd.lang.apex.ast.ApexParser.parse(ApexParser.java:51)
	at net.sourceforge.pmd.lang.apex.ApexParser.parse(ApexParser.java:37)
	at net.sourceforge.pmd.SourceCodeProcessor.parse(SourceCodeProcessor.java:113)
	at net.sourceforge.pmd.SourceCodeProcessor.processSource(SourceCodeProcessor.java:175)
	at net.sourceforge.pmd.SourceCodeProcessor.processSourceCode(SourceCodeProcessor.java:97)
	at net.sourceforge.pmd.SourceCodeProcessor.processSourceCode(SourceCodeProcessor.java:52)
	at net.sourceforge.pmd.processor.PmdRunnable.call(PmdRunnable.java:88)
	at net.sourceforge.pmd.processor.PmdRunnable.call(PmdRunnable.java:27)
	at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
	at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:514)
	at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
	at java.base/java.lang.Thread.run(Thread.java:844)
```

最後に、GitLab CI で回す方法を試行錯誤してみました。

基本的にやることはコマンドを叩くだけですが、先述の通り JRE のバージョンを気をつけることと、終了ステータスが 0 でなくても打ち切りにさせない方法、そして HTML レポートを artifact (成果物) として保存することの 3 点がミソです。

```yml
image: openjdk:8-jdk

stages:
  - pmd

pmd:
  stage: pmd
  variables:
    PMD_VAR: '5.8.1'
    PMD_OUT: 'pmd.html'
    RULESETS: 'apex-apexunit,apex-braces,apex-complexity,apex-performance,apex-security,apex-style,vf-security'
  script:
    - wget -q https://github.com/pmd/pmd/releases/download/pmd_releases%2F${PMD_VAR}/pmd-bin-${PMD_VAR}.zip
    - unzip -q pmd-bin-${PMD_VAR}.zip
    - echo `./pmd-bin-${PMD_VAR}/bin/run.sh pmd -d src -f html -R ${RULESETS}` > $PMD_OUT
  artifacts:
    paths:
      - $PMD_OUT
```

GitLab CI (他の Scriptable な CI サービスも同様) は終了ステータスが 0 か否かでステージの成功/失敗を判定し、しかも失敗の場合は後続の処理 (artifact の保存も含む) が実行されないため、何も対策しないと PMD が問題を見つけたとき (return 1 したとき) に肝心のレポートが見られなくなってしまいます。そこで、`echo` とバッククオートを使って回避してみました。`echo` した結果 (標準出力のみ) はファイルにリダイレクトしてレポートを作る仕組みです。

Salesforce DX などでビルドやテスト、デプロイのステージを設ける際も、pmd ステージはレポートを保存しなければいけませんから、問題を検出した際に CI として失敗としたい場合は一旦 pmd ステージは通過させた上で、後続のパイプラインで再度 (今度は小細工なしで) `pmd` を実行して失敗に落とすのが良さそうです。

PMD は現在 6.0 のメジャーリリースに向けて開発が進んでいます。幾つかのルールセットが細分化され、かわりに vf-security などが deprecated になるようです (後方互換のためにしばらく残るようですが)。5.x 系も盛んにアップデートされているため、今後も開発の行方を追っていきたいところです。

#### 参考文献

1. [PMD &#x2013; Running PMD via command line](https://pmd.github.io/pmd-5.8.1/usage/running.html)
2. [PMD Apex &#x2013; PMD Rulesets index: Current Rulesets](https://pmd.github.io/latest/pmd-apex/rules/index.html)
3. [PMD VF &#x2013; PMD Rulesets index: Current Rulesets](https://pmd.github.io/latest/pmd-visualforce/rules/index.html)