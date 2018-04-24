---
title: "Spring Boot と Node.js の Heroku お引越し大作戦"
date: 2018-04-14T19:46:31+09:00
draft: false
categories: ["web"]
tags: [
  "heroku",
  "java",
  "spring-boot",
  "gradle",
  "javascript",
  "node.js",
  "newrelic"
  ]
---

{{<figure src="/img/logo/heroku2.png" class="center" width="50%">}}

先日、契約している VPS ([VULTR VPS 東京リージョン](https://www.vultr.com/?ref=7053029)) で動かしている自作 Web アプリたちの負荷が上がってきたことから、Heroku へのお引越しを敢行しました。この記事はその奮闘記です。
主に、設定ファイルの環境変数化、デプロイ、Deploy to Heroku ボタンの設置、New Relic APM によるパフォーマンスモニタリングの適用の4点についてのメモをまとめました。

### 1. 設定ファイルの環境変数化

皆さんの Web アプリはどのようにパラメーターを定義・記載していますか？そして、それをどうやって環境ごとに切り替えていますか？
たとえば、標準的な Web アプリでは以下のような環境があり、それぞれデータベースやログレベルなどを異なるものを指すことが多いでしょう。

- ローカル開発環境
- ローカルユニットテスト環境
- CI ユニットテスト環境
- CI インテグレーション・E2E テスト環境
- ステージング環境
- プロダクション環境

また、ステージングやプロダクションもリージョンや提供サービスのランクなどによりホストごとに環境が分かれることも多々あります。そのため、プログラムに環境依存のパラメーターをハードコーディングをするするのではなく、外だしすることはもはや定石でしょう。

しかし、問題はその方法です。例えば、 Spring Boot では多数のパラメーター供給方法があり、次の 17 段階のオーダーに基づいて実際にランタイムが採用するプロパティ (パラメーター) を決定します<sup>[参考文献1]</sup>。

1. ホームディレクトリの Devtools グローバル設定プロパティ (devtools がアクティブのとき)
2. テスト中の `@TestPropertySource` アノテーション
3. テスト中の `@SpringBootTest#properties` アノテーション属性
4. **コマンドライン引数**
5. `SPRING_APPLICATION_JSON` 内のプロパティ (環境変数かシステムプロパティのインライン JSON 埋め込み)
6. `ServletConfig` の init パラメーター
7. `ServletContext` の init パラメーター
8. `java:comp/env` の JNDI 属性
9. **Java システムプロパティ** (`System.getProperties()`)
10. **OS 環境変数**
11. `RandomValuePropertySource` が持つ `random.*` プロパティ
12. パッケージされた jar の外にあるプロファイル固有のアプリケーションプロパティ (`application-{profile}.properties` とその YAML 版)
13. パッケージされた jar の中にあるプロファイル固有のアプリケーションプロパティ (`application-{profile}.prioerties` とその YAML 版)
14. パッケージされた jar の外にあるアプリケーションプロパティ (`application.properties` とその YAML 版)
15. パッケージされた jar の中にあるアプリケーションプロパティ (`application.properties` とその YAML 版)
16. `@Configuration` クラス中の `@PropertySource` アノテーション
17. **デフォルトプロパティ** (`SpringApplication.setDefaultProperties` で指定)

パラメーターのデフォルト値を上書きする方法が 16 通りもあるんですよ！人間の勝手な都合がコンピューターにどれだけ複雑な処理を強いていることか :innocent: よくわからない人のためになるべく簡単に解説すると、デフォルト値は環境変数で上書きできるし、それをさらにシステムプロパティで上書きできるし、それをさらにコマンドライン引数で上書きできるし、そしてさらに神のツール Spring Boot Devtools は万物の破壊と創造を司ることができるのです。

私がこれまで開発した移行対象のアプリでは、設定ファイルを複数用意して実行時の引数などで切り替える方式を採用していました。それぞれこんな感じです。

Spring Boot (w/プロファイル):

| 環境 | 設定ファイル | 実行コマンド |
| --- | --- | --- |
| ローカル開発 | `src/main/resources/application.yml` | `java -jar app.jar` |
| プロダクション | `src/main/resources/application-production.yml` | `java -jar -Dspring.profiles.active=production app.jar` |

Node.js (w/[node-config](https://github.com/lorenwest/node-config)):

| 環境 | 設定ファイル | 実行コマンド |
| --- | --- | --- |
| ローカル開発 | `config/default.yml` | `node app.js` |
| プロダクション | `config/production.yml` | `NODE_ENV=production node app.js` |

しかし、Heroku (他の PaaS も同様) では OS の環境変数でパラメーターを与えることがベストプラクティスとなっています。
Heroku が生み出した Web アプリ開発の有名な方法論 **"The Twelve-Factor App"** の 3 項目目には、次のように記されています<sup>[参考文献3]</sup>。

> 環境変数は、コードを変更することなくデプロイごとに簡単に変更できる。設定ファイルとは異なり、誤ってリポジトリにチェックインされる可能性はほとんどない。また、独自形式の設定ファイルやJava System Propertiesなど他の設定の仕組みとは異なり、環境変数は言語やOSに依存しない標準である。

というわけで、これまでオンプレや IaaS で慣れ親しんできた高機能なプロファイルスイッチングと決別し、ローレベルでプリミティブな環境変数に原点回帰することになりました。

#### 1.1 Spring Boot の場合

Spring Boot のプロパティは、環境変数と相互に変換可能です。すなわち、`spring.data.mongodb.uri` というプロパティは環境変数 `SPRING_DATA_MONGODB_URI` に対応しています。よって、この仕組みを使うことでアプリ側は何も対応することなく Heroku にフィットさせることができます。

しかし、Heroku ではアドオンから提供される環境変数をアプリから利用するというパターンがあり、全ての環境変数をデベロッパー (この記事を読んでいるあなた) が決められるとは限りません。このようなケースでは、環境変数とプロパティの対応関係を何かしらの手段で定義することになります。
また、Spring Boot は前述のように `application.properties` / `.yml` とプロファイルの機構による柔軟なパラメーター制御が最初から備わっており、Spring Security や Spring Data 等の構造化されたパラメーターたちもこのファイル一つで取り回せる利便性があります。
いくら環境変数絶対な Heroku といえども、この仕組みはなかなか手放したくないものです。
そこで便利なのが、`application.propeties` / `.yml` の中から環境変数を刺す機能を用いた、プロパティ・プロファイル・環境変数の合わせ技です。

```properties
spring.data.mongodb.uri=${MONGODB_URI}
```

```yaml
spring:
  data:
    mongodb:
      uri: ${MONGODB_URI}
```

パラメーターからパラメーターを参照しちゃうなんて、実にけしからん機能ですね :heart_eyes:
さらにすごいことに `${KEY:foo}` のようにデフォルト値を指定することもできてしまいます。
ローカル用はデフォルト値、Heroku 用は環境変数としておけば、1つの `application.yml` を七変化させることができます。

従来通りプロファイル分けも併用する場合は、どっちをローカル、どっちを Heroku (環境変数) に寄せるか検討する必要があります。
例えば、いかのようなパターンが考えられます:

- `application.yml` はローカル用かつ Git は ignore する、Heroku は `application-production.yml` で igonre しない
- `application.yml` は Heroku 用かつ Git は ignore しない、ローカルは `application-development.yml` で igonre する

起動時のプロファイル指定を開発環境か Heroku かどっちでやりたいかで決まります (両方でも良い)。

#### 1.2 Node.js の場合

Node.js では、Spring Boot のような重厚長大な設定機能はありません。環境変数をファイル化するシンプルなツール [dotenv](https://www.npmjs.com/package/dotenv) を用いて、ローカルではファイルを参照、Heroku では環境変数を参照と透過的に行うことを目指してみようと思います。
早速アプリケーションに導入してみましょう。

```
npm i --save dotenv
```

プロジェクトルートに `.env` というファイルを用意し、`KEY=VALUE` を並べていきます。こんな感じです。

```properties
LDAP_URI="ldap://foo.bar.com:389"
MONGODB_URI="mongodb://user:password@foo.bar.com:27017/database"
```

もちろんリポジトリからは ignore します。

```
echo .env >> .gitignore
```

:bulb: アプリが要求する環境変数の一覧の自己文書化は後述の「Deploy to Heroku ボタン」のセクションで登場する `app.json` にて紹介します。

アプリケーションから設定値を読み出すときは、はじめに `require` し、その後 Node.js の `process.env` から引っ張ってきます。

```js
require('dotenv').config();
console.log(process.env.LDAP_URI);
```

`require` する代わりに外部ツール ([node-foreman](https://github.com/strongloop/node-foreman)) で環境変数に読み込むこともできます。

```
npm i -g foreman
nf run node app.js
```

なお、　foreman の場合は `.env` の中身を以下のような JSON フォーマットでも記述できるようです。

```json
{
    "ldap": {
        "uri": "ldap://foo.bar.com:389"
    },
    "mongodb": {
        "uri": "mongodb://user:password@foo.bar.com:27017/database"
    }
}
```

これで、長年お世話になった node-config とはお別れです。

### 2. デプロイ

パラメーターの戦略が決まったら、次はいよいよデプロイです。

#### 2.1 Spring Boot

Spring Boot の場合はその前に一仕事。Heroku のドキュメントに従い、必要に応じて `Procfile` と `system.properties` を用意します<sup>[参考文献5]</sup>。

プロジェクトルートに以下のような `Procfile` を配置します。

```
web: java -jar -Dspring.profiles.active=production build/libs/app.jar --server.port=$PORT
```

少し補足です。

- 冒頭の `web:` は Web Dyno であることを指しています。Worker Dyno なら `worker:` で始まります<sup>[参考文献7]</sup>。
- `spring.profiles.active` は前述の環境変数入りプロファイルを指定するために書いていますが、指定しない場合は `-D` ごと削除してください。なお、`SPRING_PROFILES_ACTIVE` 環境変数を用いて、この指定自体を外出しすることも可能です。
- jar ファイルの名前を固定するには、ビルドツールにそのように指示しなければなりません。Gradle の場合は、以下のように記述します:

```groovy
bootJar {
    baseName = 'app'
    archiveName = baseName + "." + extension
}
```

`system.properties` のほうは、Java のバージョンを指定するために利用します。こんな感じです。

```properties
java.runtime.version=1.8
```

Heroku は最新の Java の追従がとても速く (Go 等他の言語もめっちゃ速い！)、(この記事を書いている今) 出たばかりの OpenJDK 10 も選べます (`java.runtime.version=10`)。
さらにすごいことに、 OpenJDK だけでなく Azul Zulu JDK も選ぶことができます。ただしこちらは現時点では 9.0.1 が最新でした (`java.runtime.version=zulu-9.0.1`)。

#### 2.2 Node.js

Node.js (npm) の場合は、今の所何もしなくても良さげです。Heroku が振るランダムなポート番号も勝手に Node.js に渡ります。

#### 2.3 デプロイ

heroku-cli を使うのが簡単です。以下のページからダウンロード、あるいは Homebrew 等を用いた方法が記載されています。

> Heroku CLI | Heroku Dev Center
>
> https://devcenter.heroku.com/articles/heroku-cli

導入したら、あとはコマンド操作で。

```
heroku login
git status # コミット漏れがあればここでコミットしておく
heroku create APP_NAME
git push heroku master
heroku open
```

:warning: 世の中には heroku create 時に `APP_NAME` が省略されているチュートリアルが山のようにありますが、これを指定しないとあなたが大事に育てた大切なアプリにランダム数字入りのおぞましい DQN ネームをつけられてしまいます。

手作業で環境変数を足していくにも、heroku-cli が便利です。

```
heroku config:set KEY=VALUE
```

:warning: 環境変数に変更があると、Heroku は Dyno を (勝手に) 再起動します。ご注意ください。

アドオンの導入もおてのもの。次は無料の MongoDB をアタッチして配備されたエンドポイントを調べるコマンドです。

```
heroku addons:create mongolab:sandbox
heroku config:get MONGODB_URI
```

:warning: アドオンは無料・有料に関わらず予めクレジットカードの登録が必要です。

無事にアプリが動作しましたか？おめでとうございます :tada:

### 3. Deploy to Heroku ボタン

OSS で Heroku Ready だよとアピールしたり、異なる Heroku 環境へのデプロイ作業を楽にしたり、環境変数を自己文書化したりするために `app.json` と Deploy to Heroku ボタンを設置してみましょう。

![Deploy](https://www.herokucdn.com/deploy/button.svg)

※ :point_up:これは押せませんよ！

`app.json` の例:

```json
{
  "name": "My App",
  "description": "Sample Application",
  "repository": "https://github.com/mikan/mikan.github.io",
  "logo": "https://github.com/mikan.png",
  "keywords": [
    "node",
    "express"
  ],
  "addons": [
    {
      "plan": "mongolab:sandbox"
    }
  ],
  "env": {
    "LDAP_URI": {
      "description": "LDAP server URL",
      "value": "ldap://foo.bar.com:389"
    },
    "LDAP_BIND_DN": {
      "description": "LDAP bind DN",
      "value": "cn=foo,dc=bar,dc=com"
    },
    "LDAP_BIND_CRED": {
      "description": "LDAP bind credential"
    },
    "LDAP_SEARCH_BASE": {
      "description": "LDAP search base",
      "value": "dc=bar,dc=com"
    },
    "LDAP_SEARCH_FILTER": {
      "description": "LDAP search filter",
      "value": "(uid={{username}})"
    },
    "MONGODB_COLLECTION": {
      "description": "MongoDB collection"
    }
  }
}
```

`"env"` の環境変数に定義できる値は、以下のものがあります<sup>[参考文献9]</sup>:

- `description`: 人間が読める説明文書
- `value`: デフォルト値 (文字列)
- `required`: 必須かオプションかを指定 (デフォルトは `true`)
- `generator`: ランダムな文字列を生成 (現状 `"secret"` のみ利用可能)

デフォルト値は、本来の (変えたいなら変えれば的な) 意味のデフォルト値として利用するのはもちろん、プレースホルダー的にサンプル値を入れてみるのも良いかもしれません。これをちゃんと書いておけば、前述の通り自己文書化にもなり、メンテナンスする人の助けになるでしょう。
あとで環境変数を足したり削ったり仕様を変えたりした場合、ここ (`app.json`) もしっかりアップデートすることをお忘れなく。

お目当のボタンのほうは、実はすごく簡単です。なんと、載せたいところに以下のように固定の画像を刺すだけです:

Markdown:

```md
[![Deploy](https://www.herokucdn.com/deploy/button.svg)](https://heroku.com/deploy)
```

HTML:

```html
<a href="https://heroku.com/deploy">
  <img src="https://www.herokucdn.com/deploy/button.svg" alt="Deploy">
</a>
```

HTTP の `referer` でどこでボタンを押したのかを特定する仕組みです。

GitHub のプライベートリポジトリの場合は `referer` は送られないため、次のように `template` パラメーターを添えます。

```md
[![Deploy](https://www.herokucdn.com/deploy/button.svg)](https://heroku.com/deploy?template=https://github.com/heroku/node-js-sample)
```

```html
<a href="https://heroku.com/deploy?template=https://github.com/heroku/node-js-sample">
  <img src="https://www.herokucdn.com/deploy/button.svg" alt="Deploy">
</a>
```

`referer` を潰すセキュリティソフトやプロキシ等があることから、パブリックリポジトリでも指しておいたほうが安全かもしれません。

### 4. New Relic APM によるパフォーマンスモニタリング

最後はおまけセクション、New Relic APM によるパフォーマンスモニタリングです。
Heroku Add-ons から簡単に使え、アプリ自体にクライアントを組み込むことで、ログだけではわからない詳細な分析ができるようになっているのが特徴です。

{{<figure src="/img/logo/newrelic_apm.png" class="center" width="50%" link="https://elements.heroku.com/addons/newrelic">}}

~~某英国の半導体設計会社の旧ロゴそっくり...~~

さて、今度はどっちみちモニタリング画面がブラウザなので、コマンドではなく画面操作でアドオンを追加してみます。

> New Relic APM - Add-ons - Heroku Elements
>
> https://elements.heroku.com/addons/newrelic

右上の [Install New Relic APM] ボタンをクリックし、対象のアプリとプランを選びます。

:bulb: New Relic APM は無料では機能がだいぶ限られます。最初の2週間は色々使えるので、この間に威力をチェックして課金するか無課金継続か検討してみましょう。

導入したら、ダッシュボードのアプリのページから New Relic APM を探してクリックします。

{{<figure src="/img/ss/heroku1.png" class="center">}}

登録した直後は SSO がうまく連携できず、認証エラーになることがありました。しばらく待つと解決します。

さて、めでたく開けてもまだデータがなにもありません。アプリから送ってやらんといかんのです。そして Spring Boot と Node.js でありえないぐらいに手順が異なります。

#### 4.1 Spring Boot

先ほど開いた New Relic の画面から、右上のアカウントドロップダウンを開き、 [Account Settings] を開きます。
すると右側に "Update your New Relic agent" というペインがあるので、そこから Java 用の Agent のバージョンをクリックするとダウンロードがはじまります。私がダウンロードしたのは `newrelic-java-4.0.0.zip` で、 9.3MB ありました。

{{<figure src="/img/ss/newrelic1.png" class="center">}}

ダウンロードした zip を展開し、 `newrelic.jar` と `newrelic.yml` をプロジェクトルートに配置します。
この方法が気持ち悪いと思われる方のために、Maven ダウンロードさせて使う方法が公式ドキュメントにありますが、コピペの 5 倍ぐらい面倒なのと、私が Maven を使っていないので割愛します。

`newrelic.yml` を少しだけ修正します。

```diff
-  license_key: '<%= license_key %>'
+  #license_key: '<%= license_key %>'

-  app_name: My Application
+  app_name: my-special-app
```

誰もライセンスキーを Git にコミットなどしたくありません！
大丈夫、Heroku は Add-ons として追加した際に環境変数にライセンスキーが刺さっています。ここに、Add-ons 経由で利用するメリットがあります (SSO でログインできるだけじゃあないんだよ)。

最後におまじないを指しておしまいです。「デプロイ」のセクションで登場した `Procfile` を次のように編集します。

```diff
- web: java -jar -Dspring.profiles.active=production build/libs/app.jar --server.port=$PORT
+ web: java -javaagent:newrelic.jar -jar -Dspring.profiles.active=production build/libs/app.jar --server.port=$PORT
```

(`-javaagent:newrelic.jar` を追加しています)

コミットして `git push heroku master` すると、New Relic APM へ計測結果が送られるようになります。

#### 4.2 Node.js

Node.js の場合は、npm で入れてコードにチョイ足しするだけで組み込むことができます。

```
npm i --save newrelic
```

私が実行した際は、 `package.json` に 4.0.0 が入りました。

次に、設定ファイルを持ってきます。

```
cp node_modules/newrelic/newrelic.js .
```

次のように修正します。

```diff
-  app_name: ['My Application'],
+  app_name: ['my-special-app'],

-  license_key: 'license key here',
+  license_key: process.env.NEW_RELIC_LICENSE_KEY,
```

アプリケーションのエントリーポイントとなるところ (`app.js`) の頭に `require` を足します。

```
require('newrelic');
```

ただし、このままだと `NEW_RELIC_LICENSE_KEY` 環境変数がないローカルで動きません。
というわけでやっつけ解決策です。

```
if (process.env.NEW_RELIC_LICENSE_KEY) {
    require('newrelic');
}
```

#### おまけ: New Relic Synthetics

New Relic には APM 以外にもいくつかの機能があり、無料で使える便利なものに Synthetics の ping 監視機能があります。

メニューから [SYNTHETICS] を選び、Monitor type を Ping に、名前と叩く URL とどっから叩くか指定し、最後に・・・

{{<figure src="/img/ss/newrelic2.png" class="center">}}

何分おきに叩くのか指定します (ここでは 15 分)。こうすることで、監視するついでに Free Dyno が 30 分で寝落ちしないように殴り続けることが可能です。APM とセットで運用なら定点観測もできて良いことづくめですね。

### まとめ

Heroku は 12 Factor に従った Web アプリを運用するためのプラットフォームなので、12 Factor を考慮していないレガシーな Web アプリケーションは例え動いたとしても Heroku の持つ機能を活かすことはできません。
しかし一度 12 Factor の "設定" (3 項目目) に従うようアプリケーションを構成すれば、少なくとも Heroku においては異なる環境に簡単にデプロイしたり、Add-ons で刺したデータストアや追加機能を環境変数を通じて瞬時にインテグレートできるようになります。
もっとも、レガシーな Web アプリを Heroku フレンドリーな Web アプリに仕立てるには他にもやることがたくさんあるのは言うまでもありません。

Heroku には他にも様々な機能や Add-ons があるので、どんどん活用していきたいところですね。

Stay tuned & Happy hacking!

### 参考文献

1. [24. Externalized Configuration](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-external-config.html)
2. [lorenwest/node-config: Node.js Application Configuration](https://github.com/lorenwest/node-config)
3. [The Twelve-Factor App （日本語訳）](https://12factor.net/ja/config)
4. [strongloop/node-foreman: A Node.js Version of Foreman](https://github.com/strongloop/node-foreman)
5. [Deploying Spring Boot Applications to Heroku | Heroku Dev Center](https://devcenter.heroku.com/articles/deploying-spring-boot-apps-to-heroku)
6. [Heroku Java Support | Heroku Dev Center](https://devcenter.heroku.com/articles/java-support)
7. [Process Types and the Procfile | Heroku Dev Center](https://devcenter.heroku.com/articles/procfile)
8. [Creating a &#39;Deploy to Heroku&#39; Button | Heroku Dev Center](https://devcenter.heroku.com/articles/heroku-button)
9. [app.json Schema | Heroku Dev Center](https://devcenter.heroku.com/articles/app-json-schema)
10. [Introduction to New Relic for Java | New Relic Documentation](https://docs.newrelic.com/docs/agents/java-agent/getting-started/introduction-new-relic-java)
11. [Java agent and Heroku | New Relic Documentation](https://docs.newrelic.com/docs/agents/java-agent/heroku/java-agent-heroku)
12. [マイクロサービスアーキテクチャ](https://amzn.to/2HBw7BA) Sam Newman (著), 佐藤 直生 (監修), 木下 哲也 (翻訳)
13. [プロダクションレディマイクロサービス ―運用に強い本番対応システムの実装と標準化](https://amzn.to/2HAzOaI) Susan J. Fowler (著), 佐藤 直生 (監修), 長尾 高弘 (翻訳)
