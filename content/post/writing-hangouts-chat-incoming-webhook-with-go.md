---
title: "Hangouts Chat の Incoming Webhook を Go で叩く"
date: 2018-03-15T22:41:43+09:00
draft: false
categories: ["web"]
tags: [
  "gsuite",
  "golang",
  "hangouts-chat",
  "mail",
  ]
---

{{<figure src="/img/logo/hangouts-chat.png" class="center">}}

今月頭の 3月1日、Google が [Hangouts Chat](https://gsuite.google.com/products/chat/) の正式版をアナウンスしました。
IT 系ニュースメディアは早速「Slack のライバル」などとセンセーショナルに報じています<sup>[参考文献1]</sup>。
ですが、Hangouts Chat はあくまで G Suite の一機能なので、独立したチャットサービスである Slack とはターゲットは幾分か異なるはずです。しかし機能強化を続けていけばいずれ Slack に追いつき追い越すことも容易に想像できます。

と、期待の膨らむ Hangouts Chat ですが、私の手元では 8 日になってようやくアクセスできるようになりました。
早速使ってみるとなるほど、 Bot や Webhook など Slack ライクな機能が備わっています。しかし Slack と比べると圧倒的な機能不足を感じます。例えば、

1. 組織外のユーザーと会話できない (これは致命的)
2. Workspace の概念がない (「チャットルーム」を束ねる上位概念がない)
3. 絵文字リアクション機能がない (かなしい)
4. カスタム絵文字がない (超かなしい)

といったところです。でも一方、Hangouts Chat ならではのメリットも確かにあります。例えば、

1. Google クオリティ (無駄がなく統一感のある UI デザインやアプリ・サービスとしての完成度の高さ)
2. DM は既存の Hangouts とログを共有、従来の Hangouts と会話可能 (ただし従来のグループチャットは統合されない)
3. Google Drive のファイルのアタッチが入力欄からダイレクトに可能、変更通知も Bot 経由で受け取れる
4. そもそも G Suite を導入済の組織にとってはコスト面も管理面も一体化できて最高の選択肢

などが挙げられると思います。

今回の記事の主題である Webhook はというと、投げ方までは Slack とほとんど同じです。
ただし URL の発行までのステップが Slack と異なる (というか劇的に簡単) ことと、フォーマットが違うのが Slack と異なるところです。

### Webhook URL 取得

そもそもの Hangouts Chat へのアクセス手段ですが、Slack 同様 Web からアクセスする方法と、アプリからアクセスする方法の二つがあります。
Web からアクセスする場合は [chat.google.com](https://chat.google.com/) というシンプルな URL をブラウザに叩きこむだけです。
アプリからアクセスする場合は [get.google.com/chat](https://get.google.com/chat/) と同じくシンプルな URL からダウンロードに進めます。
今の所、Windows, macOS, Android, iOS 版があります。
Web アプリは今流行りのレスポンシブデザインですね (もしかして PWA?)。そしてアプリ版もいわゆる "ガワネイティブ" らしく、中身は Web 版となんら代わりありません。

Hangouts Chat を開いたら、まず「チャットルーム」を作成します。Slack でいう channel です。

{{<figure src="/img/ss/hangouts-chat1.png" class="center" title="チャットルームの作成">}}

作成できましたか？試しに @Giphy で遊んでみましょう (脱線)。

{{<figure src="/img/ss/hangouts-chat2.png" class="center" title="@Giphy で遊ぶ">}}

なんだこれ。

お次に [ルーム名 👤 n] となっているところをクリックし、 [Webhook を設定] を選びます。

{{<figure src="/img/ss/hangouts-chat3.png" class="center" title="Webhook を設定">}}

着信 Webhook というダイアログが出るので、 [WEBHOOK を追加] を選ぶと、名前とアイコンを選ぶダイアログが表示されるので、適当な値を記入します。

{{<figure src="/img/ss/hangouts-chat4.png" class="center" title="着信 Webhook">}}

💡 オマケ: GitHub の機能で `github.com/<USER or ORG>.png` という URL で対象のユーザーや Organization のアイコン画像を取得できます。
手っ取り早くアイコン画像を得たいときにとても便利です。
さらにクエリパラメーター `size` でサイズを px 単位で指定することもできる超絶便利なおまけつきです。
上記の例では `https://github.com/golang.png?size=128` と入力しています。
なお、この機能は 302 リダイレクトされるため `curl` では `-L` を追加する必要があります。

Webhook URL が発行されたらコピーしておきます。

{{<figure src="/img/ss/hangouts-chat5.png" class="center" title="着信 Webhook">}}

### Go による実装

Webhook の実態は例によって JSON を HTTP POST するだけの簡単なものです。従って Go 言語ならば標準ライブラリのみで簡潔に書くことができます。
用いる標準ライブラリは `encoding/json`, `net/http`, そしてバイトスライスの `Reader` のためにちょこっと `bytes` パッケージを使うだけです。以下のコードはエラー処理とエラー時のレスポンス読み込みまで書いているので `log` と `io/ioutil` も使っています。

```go
package main

import (
	"bytes"
	"encoding/json"
	"io/ioutil"
	"log"
	"net/http"
)

const webhook = "https://chat.googleapis.com/v1/spaces/..."

func main() {
	payload, err := json.Marshal(struct {
		Text string `json:"text"`
	}{
		Text: "てすと！",
	})
	if err != nil {
		log.Fatal(err)
	}
	resp, err := http.Post(webhook, "application/json; charset=UTF-8", bytes.NewReader(payload))
	if err != nil {
		log.Fatal(err)
	}
	if resp.StatusCode != 200 {
		body, err := ioutil.ReadAll(resp.Body)
		if err != nil {
			log.Fatal(err)
		}
		log.Fatalf("HTTP %d: %s", resp.StatusCode, body)
	}
}
```

実行すると、Hangouts Chat には以下のように表示されます。

{{<figure src="/img/ss/hangouts-chat6.png" class="center" title="Webhook 結果">}}

ここまでは webhook の URL 以外は Slack と何も変わりません。

### メンションとデコレーション

メンションやテキストの飾り付けは、Slack と異なっており、以下のような書式で指定する必要があります<sup>[参考文献2]</sup>。
Markdown とは一部異なるので注意してください。

| 書式 | 記号 | 例 | 結果 |
| --- | --- | --- | --- |
| 太字 | * | &#042;hello&#042; | **hello** |
| 斜体 | _ (アンダースコア) | &#095;hello&#095; | _hello_ |
| 取り消し線 | ~ | &#126;hello&#126; | ~~hello~~ |
| 等幅文字列 | &#096; (バッククオート) | &#096;hello&#096; | `hello` |
| 等幅文字列ブロック | &#096;&#096;&#096; (バッククオート3つ) | &#096;&#096;&#096;<br/>Hello<br/>World<br/>&#096;&#096;&#096; | <pre>Hello<br/>World</pre> |
| リンク | &lt; &#124; &gt; | &lt;`https://mikan.github.io/`&#124;`my link text`&gt; | [my link text](https://mikan.github.io) |
| 全体メンション | &lt; / &gt; | &lt;users/all&gt; | @all (メンション) |
| ユーザーメンション |  &lt; / &gt; | &lt;users/123456789012345678901&gt; | @ユーザー (メンション) |

全体メンションを人間がやるときは「@全員」なのですが、API だと「@all」です。またユーザー宛メンションで用いるユーザー ID は着信メッセージの `sender` フィールドから取れとドキュメントにあります。インタラクティブな Bot を作るときに使うようです。

### カードメッセージの送信

Hangouts Chat の Webhook には Card Message というフォーマットも規定されてみました<sup>[参考文献3]</sup>。こちらも投げてみます。
先に動かした結果から見てみましょう。次の画像はドキュメントにあるピザ配達のメッセージをそのまま再現したものです (地図の座標は省略されていたので代わりに多摩川を映してみる)。

{{<figure src="/img/ss/hangouts-chat7.png" class="center" title="Card Message">}}

画像があり、タイトルとサブタイトルがあり、注文番号と状態を示すセクションがあり、地図を示すセクションがあり、そして注文を開くボタン？があります。
これらのコンポーネントは全て JSON で定義されたものです。
アイコンに関しては、上記例は画像を直接刺していますが、ビルトインのアイコンもいくつか定義されています。

さて、これを Go 言語で扱うのは簡単ではありません。
ですが一度 JSON の仕様を構造体に定義してしまえば、あとは Go が誇る構造体リテラルを使ってデータを組み立てていくだけです。

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io/ioutil"
	"log"
	"net/http"
)

const webhook = "https://chat.googleapis.com/v1/spaces/..."

type Cards struct {
	Cards []Card `json:"cards,omitempty"`
}

type Card struct {
	Header   *Header   `json:"header,omitempty"`
	Sections []Section `json:"sections,omitempty"`
}

type Header struct {
	Title      string `json:"title,omitempty"`
	Subtitle   string `json:"subtitle,omitempty"`
	ImageURL   string `json:"imageUrl,omitempty"`
	ImageStyle string `json:"imageStyle,omitempty"`
}

type Section struct {
	Header  string   `json:"header,omitempty"`
	Widgets []Widget `json:"widgets,omitempty"`
}

type Widget struct {
	KeyValue      *KeyValue `json:"keyValue,omitempty"`
	Image         *Image    `json:"image,omitempty"`
	Buttons       []Button  `json:"buttons,omitempty"`
	TextParagraph string    `json:"textParagraph,omitempty"`
}

type KeyValue struct {
	TopLabel         string   `json:"topLabel,omitempty"`
	Content          string   `json:"content,omitempty"`
	Icon             string   `json:"icon,omitempty"`
	ContentMultiLine string   `json:"contentMultiline,omitempty"`
	BottomLabel      string   `json:"bottomLabel,omitempty"`
	OnClick          *OnClick `json:"onClick,omitempty"`
	Button           *Button  `json:"button,omitempty"`
}

type Image struct {
	ImageURL string   `json:"imageUrl,omitempty"`
	OnClick  *OnClick `json:"onClick,omitempty"`
}

type Button struct {
	TextButton  *TextButton  `json:"textButton,omitempty"`
	ImageButton *ImageButton `json:"imageButton,omitempty"`
}

type TextButton struct {
	Text    string   `json:"text,omitempty"`
	OnClick *OnClick `json:"onClick,omitempty"`
}

type ImageButton struct {
	IconURL string   `json:"iconUrl,omitempty"`
	Icon    string   `json:"icon,omitempty"`
	OnClick *OnClick `json:"onClick,omitempty"`
}

type OnClick struct {
	OpenLink *OpenLink `json:"openLink,omitempty"`
}

type OpenLink struct {
	URL string `json:"url,omitempty"`
}

func main() {
	msg := Cards{[]Card{{
		&Header{
			Title:    "Pizza Bot Customer Support",
			Subtitle: "pizzabot@example.com",
			ImageURL: "https://goo.gl/aeDtrS",
		},
		[]Section{
			{
				Widgets: []Widget{
					{
						KeyValue: &KeyValue{
							TopLabel: "Order No.",
							Content:  "12345",
						},
					},
					{
						KeyValue: &KeyValue{
							TopLabel: "Status",
							Content:  "In Delivery",
						},
					},
				},
			},
			{
				Header: "Location",
				Widgets: []Widget{{
					Image: &Image{
						ImageURL: "http://maps.google.com/maps/api/staticmap?center=35.5872872,139.667575&zoom=17&size=400x300",
					},
				}},
			},
			{
				Widgets: []Widget{{
					Buttons: []Button{{
						TextButton: &TextButton{
							Text:    "OPEN ORDER",
							OnClick: &OnClick{&OpenLink{"https://mikan.github.io/"}},
						},
					}},
				}},
			},
		},
	}}}

	payload, err := json.Marshal(msg)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(string(payload))
	resp, err := http.Post(webhook, "application/json; charset=UTF-8", bytes.NewReader(payload))
	if err != nil {
		log.Fatal(err)
	}
	if resp.StatusCode != 200 {
		body, err := ioutil.ReadAll(resp.Body)
		if err != nil {
			log.Fatal(err)
		}
		log.Fatalf("HTTP %d: %s", resp.StatusCode, body)
	}
}
```

構造体定義がずらずらある以外は、そんなに複雑ではないことに気づくでしょう。
なお、使わない (ゼロ値の) フィールドを JSON 出力しないように `omitempty` フィールドタグを添えています。
埋め込む構造体をポインタにしているのもこのためです。

また、上記例には含まれておりませんが、 Card Message では、多くの文字列フィールドは一部の HTML タグが利用できます。使えるタグは、

* &lt;b&gt; (太字)
* &lt;i&gt; (斜体)
* &lt;u&gt; (下線)
* &lt;strike&gt; (取消線)
* &lt;font color=""&gt; (文字色) ※&lt;font color=\"#ff0000\"&gt;red&lt;/font&gt; のように利用
* &lt;a href=""&gt; (ハイパーリンク)
* &lt;br&gt; (改行)

が規定されています。

### 活用例

今回、この Webhook の仕組みを用いて社内で使っているシステムの通知を Hangouts Chat に連携する機能を導入してみました。
こんなシステムです (赤字が今回の改修部分):

{{<figure src="/img/diagram/gh-id-checker.png" class="center">}}

このシステムは GitHub の会社の Organization に「私の ID を追加して！」ってお願いするための申請システムで、
Organization の社内運用ルールに適合するアカウントかどうか (2FA 有効か、会社メルアド刺さってるか等) を予め自動チェックするのが目的です。

もちろん Go で実装しており、通知の本文はこんな感じです:

```go
msg := client.Message{
	Text: fmt.Sprintf("<users/all> GitHub ユーザー <https://github.com/%s|%s> から登録依頼が来ました。\n<https://github.com/orgs/%s/people|メンバー管理はこちら>", userData.Login, userData.Login, org),
}
```

通知はこのようになります:

{{<figure src="/img/ss/hangouts-chat8.png" class="center">}}

このシステムでは同時にメールも送信しており、主にメールを見るユーザーにも通知を確実に届けます。

```go
package main

import (
	"bytes"
	"log"
	"net/smtp"
	"os"
)

func sendMail(id, org string) {
	from := os.Getenv("SEND_FROM")
	to := os.Getenv("SEND_TO")
	user := os.Getenv("SMTP_USER")
	password := os.Getenv("SMTP_PASSWORD")
	server := os.Getenv("SMTP_SERVER")
	port := os.Getenv("SMTP_PORT")
	body := bytes.NewBufferString("Subject: [GitHub/" + org + "] ID 招待依頼\r\n")
	body.WriteString("Content-Type: text/plain; charset=\"UTF-8\"\r\n")
	body.WriteString("\r\n")
	body.WriteString("次のユーザーから GitHub " + org + " 組織への招待依頼が届きました: https://github.com/" + id + "\r\n")
	body.WriteString("\r\n")
	body.WriteString("メンバー管理はこちら: https://github.com/orgs/" + org + "/people\r\n")
	auth := smtp.PlainAuth("", user, password, server)
	if err := smtp.SendMail(server+":"+port, auth, from, []string{to}, body.Bytes()); err != nil {
		log.Printf("Failed to send mail, %v", err)
	}
}
```

標準ライブラリ `net/smtp` でさくっとメールを送れるあたりも Go の魅力です。
ただし、日本語メールを送る場合は上記例のように `Content-Type: text/plain; charset=\"UTF-8\"` ヘッダーを刺すをお忘れなく。

・・・また脱線してしまいましたが、いかがでしたでしょうか。簡単に連携できて、その気になれば凝ったメッセージも送れることがお分りいただけたかと思います。
Webhook で Hangouts Chat の表現力が分かってきたら、次は Bot の自作にチャレンジしたいところですね。

Stay tuned & Happy hacking!

#### 参考文献

1. [Google、Hangouts Chat、G Suite向け正式版公開――Slackのライバルを狙う  |  TechCrunch Japan](https://jp.techcrunch.com/2018/03/01/2018-02-28-hangout-chat-googles-slack-competitor-comes-out-of-beta/)
2. [Simple Text Messages &nbsp;|&nbsp; Hangouts Chat &nbsp;|&nbsp; Google Developers](https://developers.google.com/hangouts/chat/reference/message-formats/basic)
3. [Card Formatting Messages &nbsp;|&nbsp; Hangouts Chat &nbsp;|&nbsp; Google Developers](https://developers.google.com/hangouts/chat/reference/message-formats/cards)
4. [プログラミング言語Go](http://amzn.to/2HAwTNW) 第4章 - Alan A.A. Donovan (著),‎ Brian W. Kernighan (著),‎ 柴田 芳樹 (翻訳)