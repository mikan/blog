---
title: "AWS Lambda Go と Chart.js で実現する静的サイトのデータビジュアライゼーション"
date: 2018-01-23T21:19:41+09:00
draft: false
categories: ["web"]
tags: [
  "golang",
  "javascript",
  "chart.js",
  "github",
  "aws",
  ]
---

1月15日、ついに待ちに待った AWS Lambda Go がリリースされました！:tada:
今年最初のブログ記事はその活用例のご紹介をしたいと思います。

AWS Lambda は言わずと知れたサーバーレスアーキテクチャのプラットフォームですが、今まで Node.js, Java, Python, C# (.NET Core 1.0) の 4 種類のランタイムしか使えませんでした。それが今回、Go <sup>[参考文献1]</sup> と C# .NET Core 2.0 <sup>[参考文献2]</sup> に対応し、ますます強力なプラットフォームに進化しました。特に Go は、そのエコシステムがもたらすプロセスのフットプリントの小ささから、実行時間とメモリ消費でスケールする Lambda の課金体系にダイレクトに効いてくる事が大いに期待できます。そしてもちろん、AWS 公式ライブラリを始めとした膨大な Go コード資産を容易に Lambda 化出来るようになったという点も大きな魅力です。

今回は、AOSN 読書会のウェブサイト ([https://aosn.ws](https://aosn.ws)) の課題本ごとのページに設置した参加記録のチャートを動的に生成する仕組みを AWS Lambda Go と Chart.js を用いて実現することができたので紹介します。

{{<figure src="/img/ss/aosn-chart.png" class="center" width="50%">}}

こちらの元データは同じページにある参加記録の表となっています。このサイトは GitHub Pages 標準の Jekyll で生成されたもので、元データは Markdown です。読書会が終わると、議事録担当者 (だいたい私:innocent:) が今日の参加者と進捗をここに記していく運用をかれこれ 3 年以上しています。

{{<figure src="/img/ss/aosn-attends.png" class="center">}}

私は以前この Markdown をパースして Elasticsearch に送りデータビジュアライゼーションする Go 言語のコードを書いており、今回はそのコードを流用して Lambda をインプリします。Markdown の表部分に参加記録の行を足してからグラフに反映されるまでの仕組みは、次のような流れになります。

{{<figure src="/img/diagram/chartgen.png" class="center">}}

今回作るグラフは、冒頭にも示したように各回の参加者数の推移と参加者毎の参加回数のランキングの 2 つのグラフです。この流れの最終成果物 (Lambda から S3 に投入する JSON ファイル群) は次のような内容になります (課題本ごとに JSON ファイルを生成したうちの 1 つ)。

```json
{
  "times":{
    "labels":[1,2,3,4,5,6],
    "data":[4,3,3,3,3,3]
  },
  "attendees":{
    "labels":["intptr-t","mikan","budougumi0617","MrBearing","kzt-ysmr"],
    "data":[6,6,5,1,1]
  }
}
```

対応する (パース結果を流し込む) Go の構造体はこんな感じです。

```go
type DataSet struct {
	ByTimes     IntLabeledData    `json:"times"`
	ByAttendees StringLabeledData `json:"attendees"`
}

type IntLabeledData struct {
	Labels []int `json:"labels"`
	Data   []int `json:"data"`
}

type StringLabeledData struct {
	Labels []string `json:"labels"`
	Data   []int    `json:"data"`
}
```

それでは本題の AWS Lambda Go の構築手順を説明したいと思います (**前置きが長い**)。

### Handler Implementation

Go で AWS Lambda のハンドラーを実装するのは他の言語同様にとても簡単です。利用可能な関数シグニチャの一覧がドキュメントにあるので、好きなシグニチャを選んで定義するだけで、最も簡単なものは `func ()` だけです <sup>[参考文献3,5]</sup>。関数名は何でも良いです。

```
package main

import "github.com/aws/aws-lambda-go/lambda"

func HandleLambdaEvent() {
    // ...
}

func main() {
	lambda.Start(HandleLambdaEvent)
}
```

`github.com/aws/aws-lambda-go` ライブラリは、GitHub にあるので `go get` して取ってきます <sup>[参考文献4]</sup>。

デプロイ可能な zip ファイルを作るには、`GOOS=linux` でコンパイルした結果を `zip` します。Linux 持ってない？心配ありません。Go ならクロスコンパイルは鼻血がでるほど簡単です。

```
GOOS=linux go build -o main main.go
zip main.zip main
```

上記は macOS を想定したものですが、Windows で上記同様の zip を生成するために `build-lambda-zip` というツールが提供されています (`go get github.com/aws/aws-lambda-go/cmd/build-lambda-zip`)。私はこんなバッチファイルを作りました。

```
set GOOS=linux
go build -o main .
build-lambda-zip -o main.zip main
del main
set GOOS=windows
```

:warning: `GOOS` を `windows` に戻すのを忘れると悲惨なことになりますよ！

出来上がった zip はコンソールや `awscli` でアップします。ハンドラ名 (zip の中に入ってるファイル名) は、この例では `main` です。

{{<figure src="/img/ss/lambda1.png" class="center" width="80%">}}

### Webhook Integration

アップロードできたら、GitHub の Webhook をトリガーにして走らせるため、API Gateway を仕込みます。Webhook なのでメソッドは `POST` です。

{{<figure src="/img/ss/apigateway1.png" class="center" width="80%">}}

今回は Lambda プロキシ統合は使用しません。GitHub の Webhook にはただステータス 200 を返してくれさえすれば良いのです。逆に、もし Lambda Proxy でデータを返す場合はレスポンス情報を組み立てる追加の実装が必要になります (そして Malformed Lambda proxy response と戦う:innocent:)。

{{<figure src="/img/ss/apigateway2.png" class="center">}}

GitHub の Webhooks は以下のように設定します。Content type は `application/json` を選びます。これで毎回 git push されるたびに API Gateway (+ Lambda) が叩かれる仕組みが出来上がりです。

{{<figure src="/img/ss/github-webhook1.png" class="center" width="80%">}}

### Storing and Hosting

AWS Lambda Go で S3 を叩くには、すでにある `aws-sdk-go` が便利です <sup>[参考文献6]</sup>。今回書いた S3 部分のソース全文を紹介しましょう。関数 `Upload()` は文字通りファイル名と中のデータを受け取ってアップロードします。

```go
package main

import (
	"bytes"

	"github.com/aws/aws-sdk-go/aws"
	"github.com/aws/aws-sdk-go/aws/endpoints"
	"github.com/aws/aws-sdk-go/aws/session"
	"github.com/aws/aws-sdk-go/service/s3"
)

func Upload(name string, data []byte) {
	bucket := "ws.aosn.chart"
	service := s3.New(session.Must(session.NewSession(&aws.Config{
		Region: aws.String(endpoints.ApNortheast1RegionID),
	})))
	_, err := service.PutObject(&s3.PutObjectInput{
		Bucket:      aws.String(bucket),
		Key:         aws.String(name),
		Body:        bytes.NewReader(data),
		ContentType: aws.String("application/json"),
		ACL:         aws.String(s3.BucketCannedACLPublicRead),
	})
	if err != nil {
		panic(err)
	}
}
```

コードのどこにも S3 のクレデンシャル情報が現れないことに不思議に思うかもしれません。でも、このコードは確かに動きます。お察しの通り、これは `aws-sdk-go` と Lambda の Go ランタイムの組み合わせによって成し遂げられています。お見事ですね:sparkles:

お目当ての成果物が無事 S3 に上がったところで、もう一つだけ作業があります。バケットを公開することと、静的 Web サイトから直接 XHR で叩く (= ダイレクトホスティング) 用に CORS (Cross-Origin Resource Sharing) の設定を仕込むことです。

S3 のコンソール ([https://s3.console.aws.amazon.com](https://s3.console.aws.amazon.com)) で対象バケットを選び、「アクセス制限」を開くと CORS の設定が記述できるようになっています。AOSN 読書会のドメインから受け付ける設定を入れたものはこんな感じです。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<CORSConfiguration xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
<CORSRule>
    <AllowedOrigin>http://aosn.ws</AllowedOrigin>
    <AllowedOrigin>https://aosn.ws</AllowedOrigin>
    <AllowedMethod>GET</AllowedMethod>
    <MaxAgeSeconds>3000</MaxAgeSeconds>
    <AllowedHeader>*</AllowedHeader>
</CORSRule>
</CORSConfiguration>
```

これでインフラは全て整いました。

### Chart.js

ここでようやく今回のもう一つのお目当ての一つ、Chart.js の登場です。HTML5 の canvas タグを用いたリッチなチャートを平易な API で描画できる素晴らしいライブラリです。公式サイトの Samples <sup>[参考文献7]</sup> を見ると、色々な図が描画できることがわかります。

今回、Chart.js 本体は CDN を利用することにしてみました。

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/Chart.js/2.7.1/Chart.min.js"></script>
```

チャートを表示したいページに `canvas` を配置します。今回、同様のチャートを課題本ごとに描画するため、JS コードは関数にして共通化しています。

```html
### 参加者推移

<canvas id="timesChart" width="400" height="200"></canvas>

### 参加回数

<canvas id="attendeesChart" width="400" height="200"></canvas>

<script>
handleEntryCharts("1-java8");
</script>
```

共通コードはこちらです。チャートの要素に充てるカラーパレットの元データは、SAP のスタイルガイドから拝借しました <sup>[参考文献9]</sup>。ページから呼び出された `handleEntryCharts()` (一番下) で S3 に置いた JSON の XHR 取得およびパース、そしてそこから呼び出す `drawEntryCharts()` (その上) が Chart.js を描画するところです。

```html
<script>
colors = [
  'rgba(92, 186, 230, 0.7)', 'rgba(182, 217, 87, 0.7)', 'rgba(250, 195, 100, 0.7)', 'rgba(140, 211, 255, 0.7)',
  'rgba(217, 152, 203, 0.7)', 'rgba(242, 210, 73, 0.7)', 'rgba(147, 185, 198, 0.7)', 'rgba(204, 197, 168, 0.7)',
  'rgba(82, 186, 204, 0.7)', 'rgba(219, 219, 70, 0.7)', 'rgba(152, 170, 251, 0.7)'];
lineChartOptions = {
  scales: {
    yAxes: [{
      ticks: {
        beginAtZero:true
      }
    }]
  }
};
horizontalBarChartOptions = {
  scales: {
    xAxes: [{
      ticks: {
        beginAtZero:true
      }
    }]
  }
}
function drawEntryCharts(dataset) {
  var timesChart = new Chart(document.getElementById("timesChart").getContext('2d'), {
  type: 'line',
  data: {
    labels: dataset.times.labels,
      datasets: [{
        label: '参加者数',
        data: dataset.times.data,
        backgroundColor: colors
      }]
    },
    options: lineChartOptions
  });
  var attendeesChart = new Chart(document.getElementById("attendeesChart").getContext('2d'), {
  type: 'horizontalBar',
  data: {
    labels: dataset.attendees.labels,
    datasets: [{
      label: '参加回数',
      data: dataset.attendees.data,
      backgroundColor: colors
    }]
  },
  options: horizontalBarChartOptions
  });
}
function handleEntryCharts(key) {
  const request = new XMLHttpRequest();
  request.open("GET", "https://s3-ap-northeast-1.amazonaws.com/ws.aosn.chart/" + key);
  request.addEventListener("load", (event) => {
    if (event.target.status !== 200) {
      console.log(`[chartgen] ${event.target.status}: ${event.target.statusText}`);
      return;
    }
    drawEntryCharts(JSON.parse(event.target.responseText));
  });
  request.send();
}
</script>
```

描画結果 (再掲)

{{<figure src="/img/ss/aosn-chart.png" class="center">}}

### 結果

今回実装した AWS Lambda Go は GitHub から Markdown 14 個とってきてパースして集計して 1 つ 1 つ S3 にぶち込むというデモにしてはやや大きな処理ですが、それがたったの 31 MB / 670.77 ms で動きました。Java で同じ機能を実現して走らせたらまず 128MB では足りないですし、時間ももっとかかるはずです。期待通りの優れたフットプリントでした。
今回はたまたま既に書いた Go のコードを手数をかけずに Lambda 化できるようになったことがモチベーションになって手を付けましたが、スクラッチで Lambda 作る場合も積極的に Go を選んで行きたくなりますね。

今回の Lambda のコードはこちらにあります。この記事を書いたあと構成を変えていなければ、cmd/aosn2lambda ディレクトリに今回の Lambda のコードがあります。

> aosn/chartgen
>
> https://github.com/aosn/chartgen

Chart.js の関数は静的サイト、つまり AOSN 読書会ウェブサイトのリポジトリにあります。共通化した関数定義は _includes ディレクトリ内の head.html にあります。

> aosn/aosn.github.io
>
> https://github.com/aosn/aosn.github.io

参考にしてみてください。

Happy hacking!

#### 参考文献

1. [AWS Lambda での Go サポート開始 - aws.amazon.com](https://aws.amazon.com/jp/about-aws/whats-new/2018/01/aws-lambda-supports-go/)
2. [AWS Lambda での C# (.NET Core 2.0) サポート開始 - aws.amazon.com](https://aws.amazon.com/jp/about-aws/whats-new/2018/01/aws-lambda-supports-c-sharp-dot-net-core-2-0/)
3. [Programming Model for Authoring Lambda Functions in Go - AWS Lambda](https://docs.aws.amazon.com/en_us/lambda/latest/dg/go-programming-model.html)
4. [aws/aws-lambda-go: Libraries, samples and tools to help Go developers develop AWS Lambda functions.](https://github.com/aws/aws-lambda-go)
5. [AWS Lambda Go早めぐり(LambdaContext, Logging, Error...)  &middot; My External Storage](https://budougumi0617.github.io/2018/01/17/hello-aws-lambda-go/)
6. [aws/aws-sdk-go: AWS SDK for the Go programming language.](https://github.com/aws/aws-sdk-go)
7. [Chart.js | Open source HTML5 Charts for your website](http://www.chartjs.org/)
8. [Chart.js samples](http://www.chartjs.org/samples/latest/)
9. [Chart &#8211; Color Palette &#8211; Values and Names | SAP Fiori Design Guidelines](https://experience.sap.com/fiori-design-web/values-and-names/)
