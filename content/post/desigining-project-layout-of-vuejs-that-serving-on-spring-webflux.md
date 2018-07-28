---
title: "Vue.js を Spring WebFlux でホストする際のプロジェクト構成を考える"
date: 2018-07-28T19:40:00+09:00
draft: false
categories: ["web"]
tags: [
  "vue.js",
  "quasar-framework",
  "node.js",
  "gradle",
  "spring-boot",
  "kotlin"
  ]
---

2018年3月1日に Spring Boot 2.0 がリリースされてから半年がたち<sup>[参考文献1]</sup>、Kotlin の正式サポートや Spring WebFlux によるリアクティブ Web プログラミングといったパラダイムシフトが徐々に広まりをみせつつある今日このごろです。
そしてこれらバックエンド技術と、進化を続ける JavaScript フレームワークによるフロントエンド技術を組合せて Web アプリやシステムを開発するというのはよくあるパターンになっています。一方で、この組合せの境界を言語的にまたぐアプローチやフレームワークもあり、これはこれで開発チームの構成によってはすごくフィットすることがあります。

ところでフロントエンド界隈は生き残っているものでは Angular 派, React 派, Vue.js 派の 3 大派閥があり<sup>[参考文献2]</sup>、また最近では Mithril など新進気鋭なものも見かけるようになりました。
そしてそれぞれのフレームワークをベースにしたフレームワークも多数登場しており<sup>[参考文献3]</sup>、有名所では SSR (Server Side Rendering) を実現する Next.js (React ベース) や Nuxt.js (Vue.js ベース) などが挙げられるでしょう。

今回は DB 操作などを WebFlux, SPA (Single Page Application) 画面を Vue.js を用い、更に Vue.js のビルド結果のサーブにも WebFlux を用いる構成を考えてみます (もちろん、他のフレームワークにも応用が効きます)。この構成の利点は 3 つあります:

1. フロントエンドコードとバックエンドコードをひとまとめに管理できる
2. フロントエンド開発者とバックエンド開発者の双方が手軽に両者結合したテストを実行できる
3. 1つのアプリケーション (単一の jar ファイル) としてデプロイできる

欠点としては、そもそも別々にデプロイしたい (フロントのサーブとバックエンドを別々にスケールしたい) 場合にはハマらないのと、ディレクトリ構成が少し複雑になることが挙げられます。

### 基本戦略

Spring WebFlux は Gradle, Vue.js を npm でビルドし、双方のディレクトリレイアウトの慣習を一切破らずに開発したいとします。
すると問題になるのが `src` の扱いです。Gradle と npm で src 内のディレクトリ構成が全く違うので、ここは分けるのが賢明です。
もちろん `src` 改名という荒療治でも解決はできますが、ここは Gradle のサブプロジェクト機能を最大限に活用してみましょう。
概念上はこんな構成になります:

```console
app
├── backend
│   ├── gradle サブプロジェクト
│   └── src
│       └── webflux コード
├── gradle ルートプロジェクト
├── npm ファイル
└── src
    └── vue.js コード
```

それぞれ、次のような役割を振ります:

- npm: Vue.js のビルド
- gradle ルートプロジェクト: gradle-node-plugin を用いた npm ビルドの呼び出し
- gradle サブプロジェクト: WebFlux のビルド

### プロジェクトの作成

用意するものは以下の 3 つです:

- `npm` コマンド (Node.js についてくる)
- `vue` コマンド (`npm install -g @vue/cli`)
- `gradle` コマンド (Homebrew や Chocolatey で導入が楽)

:warning: Vue CLI の npm パッケージはバージョン 2 までが `vue-cli` で 3 からが `@vue/cli` です<sup>[参考文献4]</sup>。今回は 3 ベースで説明しています。

```console
$ vue create webflux-vuejs-demo
```

:bulb: このコマンドは与えられた名前でディレクトリを作り、その中にプロジェクトファイルを生成してくれます。

実行すると、次のように聞かれます。

```console
Vue CLI v3.0.0-rc.8
? Please pick a preset: 
❯ default (babel, eslint) 
  Manually select features
```

キーボードの :arrow_up: と :arrow_down: で切り替えられます。ここは default のまま Enter をターンっとタイプします。
また、 "Your connection to the default npm registry seems to be slow." と出て、別の npm リポジトリをお勧めされることがあります。私の場合、 taobao.org とか・・・中国っぽい。

実行が終わると、次のコマンドで試すよう促されます。実際に動かしてみましょう。

```console
$ cd webflux-vuejs-demo
$ npm run serve
```

ブラウザで http://localhost:8080/ にアクセスすると...

{{<figure src="/img/ss/vuejs1.png" class="center" width="80%">}}

めでたく表示されました！って、満足してはいけません。話はここからです・・・。

さきほど npm を叩いたのと同じディレクトリで、以下のコマンドを実行します。

```console
$ gradle init wrapper
```

`build.gradle` ほかいくつかのファイルが生成されました。ここで `.gitignore` に以下を追加します。

```console
# Gradle
.gradle
build
```

`build.gradle` を編集します。元からあるコメント行はいらないので消してしまいましょう。

```groovy
plugins {
    id "com.moowork.node" version "1.2.0"
}

node {
    version = '10.7.0'
    download = true
}
```

これで、 npm と同じことが gradle からできるようになりました。

| npm           | gradle                  |
| ------------- | ----------------------- |
| npm install   | ./gradlew npm_install   |
| npm run lint  | ./gradlew npm_run_lint  |
| npm run build | ./gradlew npm_run_build |
| npm run serve | ./gradlew npm_run_serve |

さらに、よく使うタスクについては、以下のように別途定義することで `./gradlew runBuild` のように呼び出せます。

```groovy
task runBuild(type: NpmTask) {
    args = ['run', 'build']
}
```

:bulb: `./gradlew tasks --all` で叩けるタスク一覧とルールを確認することができます。

### WebFlux の構成

gradle-node-plugin (com.moowork.node) の素晴らしさを体感したら、いよいよ WebFlux の構成です。

先の基本戦略に従い、 `backend` サブプロジェクトを作成します。

```
$ mkdir backend
$ vi backend/build.gradle
```

`backend/build.gradle`:

```groovy
plugins {
    id 'org.jetbrains.kotlin.jvm' version '1.2.51'
    id 'org.jetbrains.kotlin.plugin.spring' version '1.2.51'
    id 'org.springframework.boot' version '2.0.3.RELEASE'
    id 'io.spring.dependency-management' version '1.0.6.RELEASE'
}

repositories {
    jcenter()
}

dependencies {
    compile 'org.jetbrains.kotlin:kotlin-stdlib-jdk8'
    compile 'org.jetbrains.kotlin:kotlin-reflect'
    compile 'org.springframework.boot:spring-boot-starter-webflux'
    testCompile 'org.springframework.boot:spring-boot-starter-test'
}

compileKotlin {
    kotlinOptions {
        jvmTarget = '1.8'
    }   
}
compileTestKotlin {
    kotlinOptions {
        jvmTarget = '1.8'
    }   
}

bootJar {
    baseName = 'webflux-vuejs-demo'
    archiveName = baseName + '.' + extension
}
```

そしてプロジェクトルートにある `settings.gradle` に以下を追記します。

```groovy
include 'backend'
```

Router Function を実装します。Kotlin で書いてみました。

`backend/src/main/kotlin/com/github/mikan/demo/App.kt`:

```console
package com.github.mikan.demo

import org.springframework.beans.factory.annotation.Value
import org.springframework.boot.SpringApplication
import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.context.annotation.Bean
import org.springframework.core.io.Resource
import org.springframework.http.MediaType
import org.springframework.stereotype.Component
import org.springframework.web.reactive.function.server.HandlerFunction
import org.springframework.web.reactive.function.server.RequestPredicates
import org.springframework.web.reactive.function.server.RouterFunction
import org.springframework.web.reactive.function.server.RouterFunctions
import org.springframework.web.reactive.function.server.ServerRequest
import org.springframework.web.reactive.function.server.ServerResponse
import reactor.core.publisher.Mono

@SpringBootApplication
class App

fun main(args: Array<String>) {
    SpringApplication.run(App::class.java, *args)
}

@Component
class IndexHandler {
    @Value("classpath:/static/index.html")
    private lateinit var indexHtml: Resource

    @Bean
    fun indexRoutes(): RouterFunction<ServerResponse> {
        return RouterFunctions.route(RequestPredicates.GET("/"), HandlerFunction { get(it) })
    }

    fun get(request: ServerRequest): Mono<ServerResponse> {
        return ServerResponse.ok().contentType(MediaType.TEXT_HTML).syncBody(indexHtml)
    }
}
```

:bulb: Spring Boot の Welcome Page 機能により本来は自動的に `/static/index.html` が `/` にマッピングされますが、 WebFlux ではそれが実現されていません<sup>[参考文献5]</sup>。そのため参考文献に示されているワークアラウンドとして `Resource` として取得して手作業でルーティングしています。今後のアップデートで不要になる可能性があります。

### Vue.js ビルド成果物の連携

先程 Kotlin のコードに `classpath:/static/index.html` とあったように、Vue.js のビルド成果物を backend に渡さないとこのコードは動きません。
といっても、実際これを実現するのは簡単です。まずはディレクトリを作成します。

```console
$ mkdir -p backend/src/main/resources/static
```

`.gitignore` に以下を追記します。

```console
# Frontend output for backend
/backend/src/main/resources/static
```

`package.json` の `build` を次のように修正します。

```diff
-     "build": "vue-cli-service build",
+     "build": "vue-cli-service build --dest backend/src/main/resources/static",
```

これで準備ができました。いよいよビルドして実行してみましょう！

```
$ ./gradlew npm_run_build
$ ./gradlew bootRun
```

{{<figure src="/img/ss/vuejs1.png" class="center" width="80%">}}

結果は同じですね。ですが、確かに WebFlux で動いています。あとはフロントエンド・バックエンド二手に分かれてガシガシ開発するのみです。

参考までに、今回作成した一式のサンプルを GitHub に公開してみました。ご利用ください。

> https://github.com/mikan/webflux-vuejs-demo

### Vue Router の history mode の使用

Vue.js には素晴らしいルーティング機構 Vue Router が付属しており、２つの動作モード hash と history があります<sup>[参考文献6]</sup>。
hash モードは URL に /#/ を入れることで完全な URL をシミュレートしていますが、これを取り除く手段として HTML5 の History API 機能を用いた history モードが提供されています。この場合、最初の一発目だけ WebFlux が index.html をサーブし、その後は JavaScript の世界でページ遷移が実現されます (ページごとに WebFlux に取りにいきません) 。

その一発目が 404 にならないようにするには、ありったけのパスを index.html に降ってあげる必要があります。
シンプルな解決策としては、 (使わない) パスパラメーターでパターンを定義してしまうというアイデアがあります。

```console
    @Bean
    fun indexRoutes(): RouterFunction<ServerResponse> {
        val handler = HandlerFunction { get(it) }
        return RouterFunctions.route(RequestPredicates.GET("/"), handler)
                .andRoute(RequestPredicates.GET("/page"), handler)
                .andRoute(RequestPredicates.GET("/page/{sub1}"), handler)
                .andRoute(RequestPredicates.GET("/page/{sub1}/{sub2}"), handler)
    }
```

WebFlux は他の静的リソース (js や css 等) も提供しますから、トップレベルをパスパラメーターにすることはできません (例では `page` としています)。そして、他に API エンドポイントなども作ることを考えると、それらと競合しないように URL 設計する必要があります。

### Quasar Framework の使用

Vue.js ベースのフレームワークの一つに Quasar Framework (くえーさーふれーむわーく) があります<sup>[参考文献7]</sup>。
こちらで開発する際、プロジェクト作成時のコマンドと、先のビルド成果物の連携の設定手段が変わってきます。
作り方を順を追って説明します。

追加で必要なもの:

- `quasar` コマンド (`npm install -g quasar-cli`)
- `vue init` コマンド (`npm install -g @vue/cli-init`)

手順:

```
$ quasar init webflux-quasar-demo
? Project name (internal usage for dev) webflux-quasar-demo
? Project product name (official name) Quasar App
? Project description A Quasar Framework app
? Author xxx <xxx@xxx.xxx>
? Check the features needed for your project: ESLint
? Pick an ESLint preset Standard
? Cordova id (disregard if not building mobile apps) org.cordova.quasar.app
? Should we run `npm install` for you after the project has been created? (recommended) NPM
```

手順の統一のため、最後の質問を `NPM` にしています。それ以外はデフォルトのまま Enter です。

Gradle プロジェクトの構成についてはこれまでの説明と一緒ですが、ビルド成果物の出力場所の設定箇所が `package.json` から `quasar.conf.js` に変わり、次のように `build` セクションに `distDir` としてパスを追加することになります。

```diff
      ...
      build: {
+      distDir: 'backend/src/main/resources/static',
       scopeHoisting: true,
       vueRouterMode: 'history',
       ...
```

そして、これまでと同様 `./gradlew npm_run_build` でビルドするために、 `package.json` に `build` スクリプトを追加します。

```diff
   "scripts": {
+    "build": "quasar build",
     "lint": "eslint --ext .js,.vue src",
     "test": "echo \"No test specified\" && exit 0"
   },
```

これで、 `./gradlew npm_run_build bootRun` で WebFlux サーブされます。

{{<figure src="/img/ss/quasar1.png" class="center" width="80%">}}

あともうひとつ。`quasar init` コマンドは色々なファイルを生成しますが、その中に `.editorconfig` があります。
しかしこれは全てのファイルに 2 スペースインデントを強制するため、4 スペースが標準な Kotlin がひどいことになります。
そこで、以下のように修正します。

```diff
- [*]
+ [*.{js,vue}]
  charset = utf-8
  indent_style = space
  indent_size = 2
```

IntelliJ だとデフォルトで .editorconfig をサポートしているので、存在を忘れるとなぜ 2 スペースになるのかわからず混乱します :scream:

参考までに、今回作成した Quasar 版のサンプルも GitHub に公開してみました。ご利用ください。

> https://github.com/mikan/webflux-quasar-demo


Happy hacking!

#### 参考文献

1. [Spring Boot 2.0 goes GA](https://spring.io/blog/2018/03/01/spring-boot-2-0-goes-ga)
2. [kamranahmedse/developer-roadmap: Roadmap to becoming a web developer in 2018](https://github.com/kamranahmedse/developer-roadmap)
3. [25+ Best Vue.js Frameworks » CSS Author](https://cssauthor.com/vuejs-frameworks/)
4. [Overview | Vue CLI 3](https://cli.vuejs.org/guide/)
5. [Welcome page(classpath:/static/index.html) not being resolved for / uri in webflux project · Issue #9785 · spring-projects/spring-boot](https://github.com/spring-projects/spring-boot/issues/9785)
6. [HTML5 History モード | Vue Router](https://router.vuejs.org/ja/guide/essentials/history-mode.html)
7. [Quasar Framework](https://quasar-framework.org/)
