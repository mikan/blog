---
title: "AWS IoT MQTT の 443 ポートへ Go からアクセスする"
date: 2018-10-22T23:10:56+09:00
draft: false
categories: ["iot"]
tags: [
  "aws",
  "golang",
  "mqtt",
  "book",
  ]
---

{{<figure src="/img/logo/awsiot.png" class="center">}}

久々にお戯れではないホンモノの IoT の世界に戻って参りました。
大変いきいきしております。ハイ。

早速 AWS IoT × Go 言語で新シーズンの最初の記事を飾ろうと思います。

(実は前職時代に書き溜めていてまだ未公開の Spring Boot 関連の記事があるのですが、未リリースの 2.1 の新機能を使ったものなので正式版での検証を待ってからの公開とする予定です。お楽しみに。)

### 443 ポート対応

AWS IoT の MQTT は、2018年2月7日より Port 443 を介した接続をサポートしています<sup>[参考文献1]</sup>。
これはとても画期的なことです。なぜなら、モノが配備されるネットワーク環境は様々で、中には 80 (HTTP) と 443 (HTTPS) 以外は許可されていない環境もあるからです。
しかし、443 ポートで MQTT が使えるようになれば、IoT に適した軽量 Pub/Sub 通信をこうした厳しい環境にも容易に導入できます。

ただし、この 443 ポートでの MQTT には制約があり、**TLS の ALPN 拡張**をサポートしている必要があります。
ALPN 拡張は、TLS 接続上でアプリケーション層に異なるプロトコルを使うことをネゴシエーションするための拡張で、これを用いることでサーバーで一旦 HTTP 1.1 を受け入れつつ、他のプロトコルのネゴシエーションが可能になるというわけです。

{{<figure src="/img/cover/pst.jpg" class="center" width="20%" title="「プロフェッショナルSSL/TLS」にも解説あり (P51)" link="https://amzn.to/2OCHjEG">}}

AWS のブログ<sup>[参考文献2]</sup>にも解説があります。

なお、IoT プログラミングでポピュラーな Python 向けに AWS が提供しているライブラリ AWS SDK for Python<sup>[参考文献3]</sup> では、指定されたポートが 443 かどうかで分岐するコードが仕組まれており、このライブラリを使う分には ALPN について意識せずにプログラミングできます。
ただし、 Python 2.7.10 以上または Python 3.5 以上が必要という制約があります。Debian/Raspbian なら Stretch 以上です。キビシイ！

### Go からのアクセス

{{<figure src="/img/logo/golang.jpg" class="center" width="30%">}}

さて、Go 言語はどうでしょうか。
Go の ALPN サポートは標準ライブラリ `crypt/tls` にて提供されており、具体的には Go 1.4 で加わりました<sup>[参考文献4]</sup>。
これにより、何の心配もなく利用できます。

一方、Go には先程紹介したような Device SDK が提供されていないため、接続にあたっては既存のライブラリを組み合わせることになります。
具体的には以下の 2 つ。

1. AWS SDK for Go<sup>[参考文献5]</sup>
2. Eclipse Paho Go Client<sup>[参考文献6]</sup>

ところで、 Go やっていて Eclipse という固有名詞に遭遇するなんてなかなかないですね。うむ。

以下コードベースで紹介していきます。前提として以下が行われていることを想定しています。

1. `aws configure` が終わっていること
2. AWS IoT でモノ・証明書が作られ、有効化され、利用可能なポリシーがアタッチされていることと

### tls.Config

まず、ALPN 拡張を用いる上でのコード上の差異を説明します。以下が 8883 ポート時に必要な `tls.Config` 構造体です (`cert` の供給は後述)。

```go
tls.Config{
    RootCAs:            pool,
    InsecureSkipVerify: true,
    Certificates:       []tls.Certificate{cert},
}
```

次が、AWS IoT の 443 ポート時に必要な `tls.Config` です。


```go
tls.Config{
    RootCAs:            pool,
    InsecureSkipVerify: true,
    Certificates:       []tls.Certificate{cert},
    NextProtos:         []string{"x-amzn-mqtt-ca"}, // Port 443 ALPN
}
```

なんと。
たった1行増えただけで、ALPN でネゴってくれます。
こうすれば良いことに気がつくまで結構時間がかかったのはナイショです。
なお、書いてある文字列 `x-amzn-mqtt-ca` は開発者ガイドに書いてあり、次のように記されています<sup>[参考文献7]</sup>。

> ポート 443 で MQTT と X.509 クライアント証明書による認証を使用して接続を希望するクライアントは、Application Layer Protocol Negotiation (ALPN) TLS 拡張機能を実装し、ProtocolNameList で ProtocolName として x-amzn-mqtt-ca を渡す必要があります。

ここに書いてある通り、素直に Go の構造体に渡すだけだったということです。なあんだ！

### main

最後に、単体で動作する `main` 関数のあるサンプルをご紹介しておきます。

```go
package main

import (
	"crypto/tls"
	"crypto/x509"
	"fmt"
	"io/ioutil"
	"log"
	"time"

	"github.com/aws/aws-sdk-go/aws/session"
	"github.com/aws/aws-sdk-go/service/iot"
	"github.com/eclipse/paho.mqtt.golang"
)

const (
	ThingName  = "foo"
	RootCAFile = "AmazonRootCA1.pem"
	CertFile   = "xxxxxxxxxx-certificate.pem.crt"
	KeyFile    = "xxxxxxxxxx-private.pem.key"
	SubTopic   = "topic/to/subscribe"
	PubTopic   = "topic/to/publish"
	PubMsg     = `{"message": "こんにちは"}`
	QoS        = 1
)

func main() {
	// AWS IoT エンドポイントを取得
	// API を呼ぶのが面倒であれば　AWS のコンソールで得られるホストを直接使っても良い
	s := session.Must(session.NewSession())
	endpoint, err := iot.New(s).DescribeEndpoint(&iot.DescribeEndpointInput{})
	if err != nil {
		panic(fmt.Sprintf("failed to describe AWS IoT endpoint: %v", err))
	}
	log.Println("iot endpoint:", *endpoint.EndpointAddress)

	// ブローカーに接続
	tlsConfig, err := newTLSConfig()
	if err != nil {
		panic(fmt.Sprintf("failed to construct tls config: %v", err))
	}
	opts := mqtt.NewClientOptions()
	opts.AddBroker(fmt.Sprintf("ssl://%s:%d", *endpoint.EndpointAddress, 443))
	opts.SetTLSConfig(tlsConfig)
	opts.SetClientID(ThingName)
	client := mqtt.NewClient(opts)
	if token := client.Connect(); token.Wait() && token.Error() != nil {
		panic(fmt.Sprintf("failed to connect broker: %v", token.Error()))
	}
	defer client.Disconnect(250)

	// Subscribe
	log.Printf("subscribing %s...\n", SubTopic)
	if token := client.Subscribe(SubTopic, QoS, handleMsg); token.Wait() && token.Error() != nil {
		panic(fmt.Sprintf("failed to subscribe %s: %v", SubTopic, token.Error()))
	}

	// Publish
	log.Printf("publishing %s...\n", PubTopic)
	if token := client.Publish(PubTopic, QoS, false, PubMsg); token.Wait() && token.Error() != nil {
		panic(fmt.Sprintf("failed to publish %s: %v", PubTopic, token.Error()))
	}

	for {
		time.Sleep(10 * time.Second)
	}
}

func newTLSConfig() (*tls.Config, error) {
	pool := x509.NewCertPool()
	if rootCA, err := ioutil.ReadFile(RootCAFile); err != nil {
		pool.AppendCertsFromPEM(rootCA)
	}
	cert, err := tls.LoadX509KeyPair(CertFile, KeyFile)
	if err != nil {
		return nil, err
	}
	cert.Leaf, err = x509.ParseCertificate(cert.Certificate[0])
	if err != nil {
		return nil, err
	}
	return &tls.Config{
        RootCAs:            pool,
        InsecureSkipVerify: true,
        Certificates:       []tls.Certificate{cert},
		NextProtos:         []string{"x-amzn-mqtt-ca"}, // Port 443 ALPN
	}, nil
}

func handleMsg(_ mqtt.Client, msg mqtt.Message) {
	fmt.Println(msg)
}
```

なお、本コード中はリージョン指定をしていません。
Go の SDK はデフォルトで `~/.aws/config` のほうは見に行かない (`~/.aws/credentials` しか見に行かない) ので、リージョンなどを記した `config` を見に行かせるためには `AWS_SDK_LOAD_CONFIG=1` を添えてあげる必要があります。

本コードでは MQTT の Client ID に Thing Name を利用していますが、異なる値にすることもできます。
ただし、一般的なデバイスのユースケースでは Client ID に Thing Name を充てることが推奨されています<sup>[参考文献8]</sup>。

AWS IoT のコンソールのテストでサブったりパブったりしてこのコードとのやりとりを試してみてください。

{{<figure src="/img/ss/awsiot21.png" class="center" width="80%">}}

Happy hacking!

### 参考文献

1. [AWS IoT Core は、ポート 443 を使った証明書によるクライアント認証で MQTT 接続をサポートできるようになりました](https://aws.amazon.com/jp/about-aws/whats-new/2018/02/aws-iot-core-now-supports-mqtt-connections-with-certificate-based-client-authentication-on-port-443/)
2. [ポート443でTLS認証を使ったMQTT: なぜ便利で、どのように動くのか | Amazon Web Services ブログ](https://aws.amazon.com/jp/blogs/news/mqtt-with-tls-client-authentication-on-port-443-why-it-is-useful-and-how-it-works/)
3. [aws/aws-iot-device-sdk-python: SDK for connecting to AWS IoT from a device using Python.](https://github.com/aws/aws-iot-device-sdk-python)
4. [Go 1.4 Release Notes - The Go Programming Language](https://golang.org/doc/go1.4#minor_library_changes)
5. [AWS SDK for Go](https://aws.amazon.com/jp/sdk-for-go/)
6. [Eclipse Paho - MQTT and MQTT-SN software](https://aws.amazon.com/jp/sdk-for-go/)
7. [プロトコル - AWS IoT](https://docs.aws.amazon.com/ja_jp/iot/latest/developerguide/protocols.html)
8. [AWS IoT によるデバイスの管理 - AWS IoT](https://docs.aws.amazon.com/ja_jp/iot/latest/developerguide/iot-thing-management.html)
