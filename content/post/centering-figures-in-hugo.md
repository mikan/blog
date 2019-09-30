---
title: "Hugo でカスタム CSS を適用して画像の配置をイジる"
date: 2017-11-03T18:19:45+09:00
draft: false
categories: ["web"]
tags: [
  "hugo",
  ]
---

Hugo で画像を真ん中に寄せたくなったのだけど方法がわからなかったので調べてみました。

Jekyll (正確には Klamdown) の場合は、 `{:refdef}` で対象を囲むことでなんでもできちゃう汎用性があります。例えばこんな感じ:

```
{:refdef: style="float: right;"}
[![](/images/cover-microservices.jpg "マイクロサービスアーキテクチャ")](/workshop/8-microservices)
{: refdef}
```

([AOSN 読書会](https://aosn.github.io)のウェブサイトの例)

Hugo の場合、テーマとの兼ね合いになってきます。本ブログで採用しているテーマ ([hugo-bootstrap-premium](https://github.com/appernetic/hugo-bootstrap-premium)) の場合は、ソースコードを読むと `customCSS` という設定項目を読み込むように記述されているのを見つけました。

> themes/hugo-bootstrap-premium/layouts/partials/base/imports.html:
> ```
{{ range .Site.Params.customCSS }}
<link rel="stylesheet" href="{{ . | absURL}}">
{{ end }}
```

このような定義がお使いのテーマにない場合はここで一手間必要です。Hugo ではテーマを override する機構が備わっており、これを利用します。先程の例の `imports.html` に相当するファイルが同様に `themes/xxx/layouts/partials/base/imports.html` あるとしたら、それを override できるファイルのパスは `layouts/partials/base/imports.html` になります。ここにファイルをコピペして、上記の記述を差し込めば OK です。

`customCSS` を実際に指定する場所はもちろん Hugo の設定ファイルです。toml の場合はこんな感じです。

```toml
[params]
	customCSS = ["/css/custom.css"]
```

上記の例では、実際に用意するファイルのパスは `/static/css/custom.css` となります。

さて、肝心の画像を真ん中に寄せる方法ですが、Hugo の機能の一つであるショートコードの "figure" を使って CSS の class を指定してみることにしました。

```
<figure src="/img/cover/gopl_ja.jpg" class="center" link="http://www.gopl.io/" alt="プログラミング言語Go">
```

(コピペしたあと `{{` と `}}` で囲ってください)

CSS は以下のようになります。

```css
figure.center {
    text-align: center;
}
figure.center img {
    margin: 0 auto;
}
```

結果は・・・

{{<figure src="/img/ss/centering-hugo-fig.png" class="center" alt="centering" width="80%">}}

見事、真ん中になりました。

同じノリで、右寄せもできます。応用として、画像を横に並べていくこともできます。CSS クラス `inline` としたときの CSS は以下のようになります。

```css
figure.inline {
    display: inline;
}
figure.inline img {
    display: inline;
}
```

元々のテーマには手を入れずに override でピンポイントに定義を追加していける点、とても使い心地が良いですね。

#### 参考文献

1. [Hugo  | Customize a Theme](https://gohugo.io/themes/customizing/)
2. [Hugo  | Shortcodes](https://gohugo.io/content-management/shortcodes/)