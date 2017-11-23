---
title: "force-client-go: Go で Salesforce の REST API を叩く"
date: 2017-11-24T00:34:56+09:00
draft: false
categories: ["salesforce"]
tags: [
  "golang",
  "force.com",
  "github",
  ]
---

Go 言語 (Golang) で Salesforce の Force.com REST API を叩く方法と、それらを簡単に扱うためにクライアントライブラリ [force-client-go](https://github.com/mikan/force-client-go) を開発したので、そのご紹介。

{{< figure src="/img/logo/force-dot-com.png" class="center" width="50%" link="https://www.salesforce.com/jp/products/platform/products/force/" >}}

Salesforce は CRM を始めとした様々な SaaS を提供していますが、Force.com と呼ばれる一連のサービスやツールを用いると様々なアプリケーションを開発・デプロイできる PaaS にもなり、中でも REST API によるデータ操作は極めて強力な拡張・連携ポイントになります。API コール制限<sup>[参考文献3]</sup>などの考慮すべき制約はあるものの、最初から制約を考慮して設計すればたいていのユースケースはクリアできますし、いざというときは札束で殴れます:innocent:

REST API を叩くとき、最初の難関は認証です。幸い Force.com REST API は以下の 3 つの選択肢があります:

1. Web サーバーフロー
2. ユーザエージェントフロー
3. ユーザ名パスワードフロー

1番目と2番目はユーザーがブラウザでログインした結果をアプリケーションに返すフローで、コールバックを受け付けるセキュアなエンドポイント (HTTP はダメ) を建てる必要があります。コンソールアプリケーションだったり、開発中にちょこっと叩いたりするには都合が悪いですね。
従って、ここでは3番目を選びます。しかしこれはあくまで開発用、個人用アプリケーションのための選択肢であることをお忘れなく。なお、Salesforce の接続アプリケーションの設定ではコールバック URL の入力が必須項目になっていますが、3番目だけ使うならば「 https://localhost/ 」など適当なものを入れておけば大丈夫です。

{{< figure src="/img/ss/salesforce-setting1.png" class="center">}}

さて、これを Go 言語で叩く方法ですが、ベタベタ書くとこんな感じになります。

```go
package main

import (
	"encoding/json"
	"log"
	"net/http"
	"net/url"
)

func main() {
	const loginURL = "https://login.salesforce.com/services/oauth2/token"
	values := url.Values{}
	values.Add("grant_type", "password")
	values.Add("client_id", "<CLIENT_ID>")
	values.Add("client_secret", "<CLIENT_SECRET>")
	values.Add("username", "<USER_NAME>")
	values.Add("password", "<PASSWORD>"+"<SECURITY_TOKEN>")
	res, err := http.PostForm(loginURL, values)
	if err != nil {
		log.Fatal(err)
	}
	var session map[string]string
	decoder := json.NewDecoder(res.Body)
	if err := decoder.Decode(&session); err != nil {
		log.Fatal(err)
	}
	log.Printf("Login successful. Instance: %s", session["instance_url"])
}
```

補足:

- Sandbox に接続する場合は `login.salesforce.com` ではなく `test.salesforce.com` になります
- `<SECURITY_TOKEN>` はユーザメニューの設定から「私のセキュリティトークンのリセット」を選んでリセットすることで得られます

認証応答には以下の要素が含まれています<sup>[参考文献4]</sup>:

- `access_token`: アクセストークン
- `instance_url`: API コール送信先の Salesforce インスタンス URL
- `id`: 認証されたユーザの ID URL
- `signature`: `id` の SHA256 の署名
- `issued_at`: 署名作成日時
- `token_type`: `Bearer` 固定 <small>(参考文献4には記載されていないがいつも返ってくる)</small>

ここの情報を使って様々な API エンドポイントにリクエストが送れるわけです。

めでたくログインできたら、クエリを投げてみましょう。メソッドは GET ですが、ヘッダーをいくつか設定する必要があるため、便利な `http.Get(url)` は使えません (他の操作も同様)。

```go
	req, err := http.NewRequest(http.MethodGet, session["instance_url"]+"/services/data/v41.0/query", nil)
	if err != nil {
		log.Fatal(err)
	}
	req.Header.Add("Authorization", session["token_type"]+" "+session["access_token"])
	req.Header.Add("Accept", "application/json")
	values = url.Values{}
	values.Add("q", "SELECT Name FROM Contact LIMIT 3")
	log.Printf("POST %s\n", req.URL)
	req.URL.RawQuery = values.Encode()
	res, err = http.DefaultClient.Do(req)
	if err != nil {
		log.Fatal(err)
	}
	defer res.Body.Close()
	decoder = json.NewDecoder(res.Body)
	var out interface{}
	if err = decoder.Decode(&out); err != nil {
		log.Fatal(err)
	}
	log.Printf("%v\n", out)
```

補足:

- API によっては `X-PrettyPrint: 1` というヘッダーを追加することで、JSON レスポンスがインデントされて返ってくることがありますが、Query API は今のところ対応していないようです

うまく結果が返ってきましたか？おめでとうございます:tada: もし `Authorization` ヘッダがおかしかったり、そもそもリクエスト先の `instance_url` が認証応答と違っていたりすると、以下のような応答が返ってきてしまいます。その際はコードや設定を見直してみてください。

```json
{
  "message": "Session expired or invalid",
  "errorCode": "INVALID_SESSION_ID"
}
```

それにしても、`http` パッケージの簡易メソッドが使えないため、だらだらと長いコードになっていますね。リクエスト結果をデコードするのも数行に渡っていて、ちょっとつらくなってきました。構造体にデータを入れようとしたら、さらに定義もつらつら書かないといけません。さらに Go 言語 1.7 で導入された `context` パッケージも使えていません。そこで、私が開発したクライアントライブラリ [force-client-go](https://github.com/mikan/force-client-go) のご紹介です。

{{< figure src="/img/ss/github-force-client-go1.png" class="center" width="80%" title="force-client-go" link="https://github.com/mikan/force-client-go">}}

これを import すれば、先程のダラダラ Query コードがこの通り！

```
var out interface{}
client.Query(context.Background(), "SELECT Name FROM Contact LIMIT 3", &out)
log.Printf("%v\n", out)
```

ね、簡単でしょ:sparkles: しかも `context` を渡せるので、キャンセル処理を自在にコントロールできます。

現在は、先程の Query 操作ほか基本的な CRUD 操作、いくつかの SObject の struct 定義、そして簡単に試せる CLI ユーティリティとサンプルコードが付属しています。

```
// Create
contact := sobject.Contact{FirstName: "Test", LastName: "User"}
id, err := client.Create(context.Background(), sobject.ContactObjectName, &contact)

// Read
var readResult sobject.Contact
err = client.Read(context.Background(), sobject.ContactObjectName, id, &readResult)

// Update
update := sobject.Contact{FirstName: "Test2"}
err = client.Update(context.Background(), sobject.ContactObjectName, id, &update)

// Delete
err = client.Delete(context.Background(), sobject.ContactObjectName, id)
```

今後暇をみて叩けるエンドポイントや SObject 定義を追加していこうと思います (README.md の [Future works](https://github.com/mikan/force-client-go#future-works) 参照)。・・・それより先にカバレッジを上げろって？善処しまーす:innocent:

Stay tuned & Happy hacking!

#### 参考文献

1. [Force.com REST API の概要 | Force.com REST API 開発者ガイド | Salesforce Developers](https://developer.salesforce.com/docs/atlas.ja-jp.api_rest.meta/api_rest/intro_what_is_rest_api.htm)
2. [セールスフォース・ドットコム用語解説 - セールスフォース・ドットコム](https://www.salesforce.com/jp/campaign/dictionary/)
3. [API 要求の制限 | Salesforce Developer の制限クイックリファレンス | Salesforce Developers](https://developer.salesforce.com/docs/atlas.ja-jp.salesforce_app_limits_cheatsheet.meta/salesforce_app_limits_cheatsheet/salesforce_app_limits_platform_api.htm)
4. [ユーザ名パスワード OAuth 認証フローについて | Force.com REST API 開発者ガイド | Salesforce Developers](https://developer.salesforce.com/docs/atlas.ja-jp.api_rest.meta/api_rest/intro_understanding_username_password_oauth_flow.htm)
5. [状況コードとエラー応答 | Force.com REST API 開発者ガイド | Salesforce Developers](https://developer.salesforce.com/docs/atlas.ja-jp.api_rest.meta/api_rest/errorcodes.htm)
