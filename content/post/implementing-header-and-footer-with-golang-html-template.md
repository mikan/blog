---
title: "Go の html/template でヘッダーやフッター等の共通化を実現する方法"
date: 2019-12-08T01:04:20+09:00
draft: false
categories: ["web"]
tags: [
  "golang",
  ]
---

最近新人くんに Go 言語を教えているのですが、「これは確かに分かりにくいよな」って思ったところがあったので1つ取り上げて解説したいと思います。
それは、「自分で Go でテンプレートの分割をやろうとしたところ値をうまく渡せず諦めました。これ、どうやったんですか？」といった質問でした。

Go の標準ライブラリの `html/template` (`text/template` ではない) はびっくりするぐらいシンプルなのにとてもパワフルなテンプレートエンジンです。
私は現在この `html/template` を用いて 43 個もの画面を持つ Web アプリケーションの開発・運用しており、中には 1000 行を軽く超える HTML もありますが、どのページも高速でレンダリングされます。
また、本記事で紹介する部品化を使いこなせば、フロントエンドフレームワークでやっているコンポーネント化に近いこともある程度までは実現することができます。

冒頭の質問に対する回答例を紹介するまえに、まずは基本的な使い方をおさらいしましょう。

### おさらい: html/template の基本

まず、次のようにファイルを用意します。

```
.
├── server.go
└── template
    └── index.html
```

`index.html` を作りましょう。

```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>{{.Title}}</title>
</head>
<body>
<h1>{{.Title}}</h1>
<p>{{.Message}}</p>
<p>現在時刻: {{.Time.Format "2006/1/2 15:04:05"}}</p>
</body>
</html>
```

`server.go` を書きましょう。
上記のテンプレート `index.html` では `Title`, `Message` と `Time` の 3 つの値を参照していました。
値を供給してテンプレートを実行します。

```go
package main

import (
	"html/template"
	"log"
	"net/http"
	"time"
)

func main() {
	port := "8080"
	http.HandleFunc("/", handleIndex)
	log.Printf("Server listening on port %s", port)
	log.Print(http.ListenAndServe(":"+port, nil))
}

func handleIndex(w http.ResponseWriter, r *http.Request) {
	t, err := template.ParseFiles("template/index.html")
	if err != nil {
		log.Fatalf("template error: %v", err)
	}
	if err := t.Execute(w, struct {
		Title   string
		Message string
		Time    time.Time
	}{
		Title:   "テストページ",
		Message: "こんにちは！",
		Time:    time.Now(),
	}); err != nil {
		log.Printf("failed to execute template: %v", err)
	}
}
```

`handleIndex` 関数は、テンプレートをファイルから読み込んでパースし、値を詰めて実行しています。
ここではその場で無名の struct を作って渡していますが、map で渡すこともできます。ただし map の場合は value の型が揃っていないといけません (あるいは `interface{}` にしないといけません)。

`main` 関数はパスと関数をマッピングし、サーバーを起動しています。コマンド `go run server.go` で実行してみましょう。

{{<figure src="/img/ss/template-demo1.png" class="center">}}

ばっちりですね！

さて、コードを読むと (書くと) リクエストが来るたびにテンプレートファイルを読み込むのは効率が悪いと感じるでしょう (テンプレートのデバッグ中には嬉しいのですが)。
起動時に読み込むように改良してみます。

```go
// import は省略しました

var templates = make(map[string]*template.Template)

func main() {
	port := "8080"
	templates["index"] = loadTemplate("index")
	http.HandleFunc("/", handleIndex)
	log.Printf("Server listening on port %s", port)
	log.Print(http.ListenAndServe(":"+port, nil))
}

func handleIndex(w http.ResponseWriter, r *http.Request) {
	if err := templates["index"].Execute(w, struct {
		Title   string
		Message string
		Time    time.Time
	}{
		Title:   "テストページ",
		Message: "こんにちは！",
		Time:    time.Now(),
	}); err != nil {
		log.Printf("failed to execute template: %v", err)
	}
}

func loadTemplate(name string) *template.Template {
	t, err := template.ParseFiles("template/" + name + ".html")
	if err != nil {
		log.Fatalf("template error: %v", err)
	}
	return t
}
```

それっぽくなってきましたね！

それでは質問に答える時が来ました。

### ヘッダーとフッターを定義

まずはファイルを作成しましょう。
名前は `_header.html` と `_footer.html` とします。
これはあくまでプラクティスなのですが、部品であることが名前からわかるようにしたいですよね。私はシンプルにアンスコをつけることにしました。
もしかしたら、ディレクトリを分けるほうが好きな人もいるかもしれません。もちろんそれでも構いません。

```
├── server.go
└── template
    ├── _footer.html
    ├── _header.html
    └── index.html
```

部品を定義するには、 `{{define "<NAME>"}}` とします。お尻には `{{end}}` も必要です。

まずは `_header.html` から。今回は以下のようにしてみました。

```html
{{define "header"}}
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>{{.Title}}</title>
</head>
<body>
<h1>{{.Title}}</h1>
{{end}}
```

続いて、 `_footer.html` です。こちらは可変部分がない静的な部品にしてみました。

```html
{{define "footer"}}
<div>Copyright &copy; mikan</div>
</body>
</html>
{{end}}
```

元の `index.html` はどうなるのでしょうか？ここがミソです。こうなります！

```html
{{template "header" .}}
<p>{{.Message}}</p>
<p>現在時刻: {{.Time.Format "2006/1/2 15:04:05"}}</p>
{{template "footer"}}
```

header と footer の呼び出しがちょこっとだけ違うことに気が付きましたか？
先程書いた header は値を利用していました。一方 footer は値を利用していません。
値を利用する動的なテンプレートの場合は、template に値を渡してあげる必要があります。
値を利用しない静的なテンプレートの場合は、なにも渡す必要がないのです。
このちょっとした違いが、初学者のハマりポイントの1つ目だと思われます。

さて、テンプレートの準備が終わったら、 Go のコードもそれを読むように指定する必要があります。
とはいえ、直すポイントは一箇所だけです。
先程作った `loadTemplate` 関数が修正箇所になります。こうなります。

```go
func loadTemplate(name string) *template.Template {
	t, err := template.ParseFiles(
		"template/"+name+".html",
		"template/_header.html",
		"template/_footer.html",
	)
	if err != nil {
		log.Fatalf("template error: %v", err)
	}
	return t
}
```

関数 `ParseFiles()` の引数は、関数名が示す通り可変長になっていて複数のパスを受け取ります。
最上位となるテンプレートを第一引数に渡したあと、**その後ろに**部品を列挙していくのです。

実はここもハマりポイントがあります。太字で強調しましたが、 `ParseFiles()` が返すテンプレートの識別子は第一引数に渡したものが採用される仕様となっています。
なので、部品を最初に書いてはいけません。部品は最後です。なお、部品同士の順序は自由です (footer が header の前にあっても良い)。

ハマりポイントを 2 つ超えれば、晴れて部品化は完了です。もう一度プログラムを実行してみましょう。

{{<figure src="/img/ss/template-demo2.png" class="center">}}

ヘッダーに移した「テストページ」という部分と、フッターの記載内容が無事表示されました！

OK 完全に理解した！これでこの記事の役目はおしまい？このブラウザタブ閉じちゃう？まあ、それでもいいです。でももうひとつ私からアドバイスがあります！

### 部品が使うデータの分離

部品を扱えるようになったら、もう一歩先に進んでみましょう。

ページがたくさんある Web アプリケーションを想像してみてください。
ヘッダーに新たな値が欲しくなったとき、いまある全ページの struct に新しい値を詰めないとどうなりますか？もちろん壊れてしまいます。そんな Web アプリケーション、メンテしたくありませんよね。

そこで、値を使うテンプレートの部品には専用の struct と便利関数を提供し、これを解決します。

まずは header テンプレートから。 `UserName` という値を使う行を追加しました。

```html
{{define "header"}}
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>{{.Title}}</title>
</head>
<body>
<h1>{{.Title}}</h1>
<p>ようこそ {{.UserName}} さん!</p>
{{end}}
```

これに対応する struct と便利関数を Go で書きます。

```
type header struct {
	Title    string
	UserName string
}

func newHeader(title string) header {
	return header{Title: title, UserName: "ゲスト"}
}
```

この構造体を、ハンドラーから利用し、テンプレートに渡します。

```go
func handleIndex(w http.ResponseWriter, r *http.Request) {
	if err := templates["index"].Execute(w, struct {
		Header  header
		Message string
		Time    time.Time
	}{
		Header:  newHeader("テストページ"),
		Message: "こんにちは！",
		Time:    time.Now(),
	}); err != nil {
		log.Printf("failed to execute template: %v", err)
	}
}
```

`index.html` からは、ハンドラーから渡される `Header` を `header` テンプレートに渡します。

```html
{{template "header" .Header}}
<p>{{.Message}}</p>
<p>現在時刻: {{.Time.Format "2006/1/2 15:04:05"}}</p>
{{template "footer"}}
```

`{{template "header" .}}` だったところが `{{template "header" .Header}}` になりました。
これで部品の中のスコープが `header` の中になります。

実行してみると・・・

{{<figure src="/img/ss/template-demo3.png" class="center">}}

`UserName` が表示されました！

なお、今回のような変更は `index` のテンプレートを修正しないまま (`{{template "header" .}}` のままで) 適用することもできます。
その場合、部品のほうで `{{.UserName}}` ではなく `{{.Header.UserName}}` と呼び出すことになります。

### おわりに

Go の基本的なテクニックと、ちょっとした `template` のコツを組み合わせると、そこそこの規模の Web アプリケーションも難なく開発できるようになります。

また、本記事では省きますが `FuncMap` という機能があります。これを用いるとテンプレートから思い思いの Go 関数を呼び出せるようになり、表現力が倍増します。テンプレートの部品化と併せて、ぜひ習得してみてください。

すべての `html/template` の仕様は Godoc にあるので、困ったら頑張ってここを見れば最終的には答えがみつかるはずです。

> template - The Go Programming Language
>
> https://golang.org/pkg/html/template/

本記事で紹介したコードの完全版は、以下の場所にあります。ライセンスは Do What The Fuck You Want To Public License です。自由にコピペして頂いて構いません :smile:

> https://github.com/mikan/template-demo

Happy hacking!
