---
title: "oauth2_proxy と Auth0 を用いた Nginx のお手軽 OAuth 化"
date: 2018-05-23T20:42:54+09:00
draft: false
categories: ["web"]
tags: [
  "oauth",
  "openid-connect",
  "nginx",
  "auth0",
  ]
---

{{<figure src="/img/ss/authzero7.png" class="center">}}

Nginx でホストしているページや Web アプリにちょっとした認証をかけたいとき、簡単だからと BASIC 認証をかけて運用しているケースは少なくないと思います。
例えば、サーバー監視用のダッシュボードだったり、本番ではないステージングや開発環境だったり、ひみつのファイル置き場だったり。
しかし BASIC 認証は手軽な反面、気をつけて運用しないといけません。例えば、

- 可逆形式でパスワードを送信するため、少なくとも HTTPS は必須
- ログアウトができない
- ユーザーに安全にサインアップさせる仕組みを作るのが面倒

などなど。それでも使われているのは、何にも代えがたいメリット、つまり手軽に導入できることがあると思います。
なんせ、Nginx なんてたったこれだけで有効になりますから。

```nginx
auth_basic "Restricted";
auth_basic_user_file /path/to/htpasswd;
```

肝心の `htpasswd` も、ブラウザでちょちょいと生成してくれる便利なジェネレーターを使えば `apache2-utils` など取り寄せる必要すらありません<sup>[参考文献1]</sup>。

でも、同じぐらいの (とまではいかなくてもあとすこしほんのちょっとの) 手順で好きな認証サービスと自在に繋がったらかっこよくありませんか？
そんなことを考えながら色々と調べたり試したりしていたところ、サーバーサイドに仕込むツールと、クラウドサービスの組み合わせが最高だったのでご紹介。

### アーキテクチャ

今回認証を実現するのに用いるツールは短縮 URL サービスの Bitly 社が開発している [oauth2_proxy](https://github.com/bitly/oauth2_proxy) というものです。これは Nginx とは独立して (別プロセスとして) 動く Go 言語製のサーバープログラムで、Nginx の `auth_request` と連動させて利用します<sup>[参考文献2,3]</sup>。

認証サービスは包括的な認証統合サービスである [Auth0](https://auth0.com/) をチョイスします。なお、oauth2_proxy はそれ自体が Google, LinkedIn, Facebook, GitHub, Azure, GitLab を直接サポートしていますので、これ以外には絶対に手を出さん！というのならそれでも良いと思います。しかし Auth0 は一度使ってみると便利すぎて震えること間違いなしです！自前のユーザーストアはもちろん備わってますし、ソーシャルログイン 2 箇所までは無料で利用できます。しかも有効化はたったのワンクリックです。対応するサービスは Google, Facebook, Microsoft, LinkedIn, GitHub, Dropbox, BitBucket, PayPal, Twitter, Amazon, vKontakte, Yandex, Yahoo!, 37 signals, Box, Salesforce, Salesforce Community, Fitbit, Baidu, Renren, Weibo, AOL, Shopify, WordPress, Dwolla, MiiCard, Yammer, SoundCloud, Instagram, The City, Planning Center, Evernote, Exact, DoCoMo... とにかくたくさんあります！

{{<figure src="/img/diagram/nginx-oauth-auth0.png" class="center">}}

### Auth0 の準備

まずは Auth0 のアカウントを作るところから。以下のサイトからサインアップすることができます。

> Single Sign On &amp; Token Based Authentication - Auth0
>
> https://auth0.com/jp/

テナントまで作り終えたら、 "Application" を作ります。

{{<figure src="/img/ss/authzero2.png" class="center" width="80%">}}

名前を入れ、 "Regular Web Application" を選んで [Create] を押します。

{{<figure src="/img/ss/authzero3.png" class="center" width="80%">}}

アプリケーションが作成されると　Quick Start が表示されると思いますが、今回はプログラミングなど必要ありません。"Settings" タブに移動します。

{{<figure src="/img/ss/authzero4.png" class="center" width="80%">}}

"Allowed Callback URLs" に、次のような URL を入力します:

- `https://your-site.com/oauth2/callback`

{{<figure src="/img/ss/authzero6.png" class="center">}}

:bulb: 設置サイト (Nginx) で後ほど `/oauth2` 以下を oauth2_proxy ツールにプロキシするので、ここでは設置サイトのホスト名のところ (`your-site.com` と書いたところ) だけ合わせてください

以下の情報を後で用いますので、覚えておいてください (といっても暗記は無理なレベルです・・・)。

- Domain (`xxx.auth0.com`)
- Client ID (32文字)
- Client Secret (64文字)

"Settings" タブの要点を抑えたら、次は "Connections" タブを開き、このアプリケーションに適用する認証手段を選択します。
ソーシャルログインのソースは画面左側のメニューから "Connections" を選び、好きなサービスを有効化します (無料枠は 2 つまで)。

{{<figure src="/img/ss/authzero5.png" class="center" width="80%">}}

### oauth2_proxy のセットアップ

次は oauth2_proxy のセットアップです。本稿執筆時点の最新版は `2.2` なのですが、ここには `oidc` (OpenID Connect) プロバイダーがないため、master ブランチから自分でビルドします (本稿執筆時点では `2.2.1-alpha` 相当)。自分でビルドといっても、oauth2_proxy は Go 製ですので簡単です。順に説明します。

もし、Go ツールチェインを持っていなければ、何らかの手段でそれを導入します。Linux ディストリビューションのパッケージではたいてい `golang` (大多数) か、あるいは `go` (Arch 等) です。[golang.org](https://golang.org/dl/) から tar ball を落としても良いでしょう。なお、Go 1.7 の新機能 `context` パッケージを利用しているので、あまりに古いと (1.7 未満だと) ビルドは失敗します。それと、`git` コマンドも入っていなかったらこのタイミングで入れてください。

お次は `GOPATH` 環境変数の確認です。`echo $GOPATH` などと打ちパスが返ってくれば OK です。空っぽならば、この場で設定します。例えばこんな感じです。

```bash
sudo mkdir /opt/go
sudo chown <user>:<group> /opt/go
export GOPATH=/opt/go
```

それでは oauth2_proxy を導入します。

```bash
go get -u github.com/bitly/oauth2_proxy
```

エラーなく終了したら、 `$GOPATH/bin/oauth2_proxy` ができているはずです。

[公式サンプル](https://github.com/bitly/oauth2_proxy/blob/master/contrib/oauth2_proxy.cfg.example)を参考にしながら設定ファイルを作ります。ファイル名・場所はもちろん好みです。

`/etc/oauth2proxy.conf`:

```
http_address = "127.0.0.1:4180"
provider = "oidc"
oidc_issuer_url = "https://xxxx.auth0.com/"
redirect_url = "https://your-site.com/oauth2/callback"
client_id = "********************************"
client_secret = "****************************************************************"
email_domains = [
    "your-site.com"
]
cookie_secret = "secret"
cookie_secure = false
```

少し補足します。

* `oidc_issuer_url` - Auth0 の設定画面に表示されていた "Domain" の値の URL 表現が入ります
* `redirect_url` - 設置サイト (Nginx) で後ほど `/oauth2` 以下を本ツールにプロキシするので、ここでは設置サイトのホスト名のところ (`your-site.com` と書いたところ) だけ合わせてください
* `client_id` - Auth0 の設定画面に表示されていた "Client ID" の値が入ります (32文字)
* `client_secret` - Auth0 の設定画面に表示されていた "Client Secret" の値が入ります (64文字)
* `email_domains` - 認証を許可するメールアドレスのドメインを制限する場合に利用します (複数ある場合はカンマ区切りで列挙します) が、ここで制限しない場合は `"*"` となります (ここで制限するということは、Auth0 は通してもアプリケーションで拒否するケースがある、ということを意味します)

複数指定する場合:

```
email_domains = [
    "foo.com",
    "bar.com"
]
```


制限しない場合:

```
email_domains = [
    "*"
]
```

設定を終えたら、軽く喰わせてみましょう。

```
/opt/go/bin/oauth2_proxy -config /etc/oauth2proxy.conf
```

エラーはありませんか？
間違って `OAuthProxy configured for Google Client ID: xxx` などと表示されていませんか？ (※ ビルドしたバージョンに設定で指定した `oidc` がないと、デフォルトの Google モードになってしまいます)
問題なければ、Ctrl+C で殺して、ちゃんとサービスにしてあげましょう。

### oauth2_proxy の systemd 化

oauth2_proxy はサービス運用に欠かせない大事なデーモンになります。そこで systemd のサービスにしておきましょう。

`/etc/systemd/system/oauth2proxy.service`:

```
[Unit]
Description=oauth2_proxy daemon
After=syslog.target network.target

[Service]
ExecStart=/opt/go/bin/oauth2_proxy -config /etc/oauth2proxy.conf
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=always
User=www-data
Group=www-data

[Install]
WantedBy=multi-user.target
```

:bulb: `User`, `Group` やバイナリ、設定ファイルのパス (`ExecStart`) 等は適宜変えてください。

設定ファイルができたら、登録・起動します。

```bash
sudo systemctl daemon-reload
sudo systemctl enable oauth2proxy
sudo service oauth2proxy start
```

### Nginx の設定

誰だ・・・ほんのちょっとの手順だからーとか言ったやつ・・・！でも、これで最後です！いよいよ Nginx に設定を入れ込んでいきます。

まず、先ほど起動した oauth2_proxy のエンドポイントを露出させます。有効な設定ファイル (`/etc/nginx/site-enabled/default` かな？) の `server` セクション内に、以下を追加します。

```nginx
	location /oauth2/ {
		proxy_pass http://localhost:4180;
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Scheme $scheme;
		proxy_set_header X-Auth-Request-Redirect $request_uri;
	}
	location = /oauth2/auth {
		proxy_pass http://localhost:4180;
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP: $remote_addr;
		proxy_set_header X-Scheme $scheme;
		proxy_set_header Content-Length "";
		proxy_pass_request_body off;
	}
```

:warning: 1つ目が `location /...` で2つ目が `location = /...` なので注意！

次に、認証を有効化したい `location` (例では `/secret`) に以下を追加します。

```nginx
	location /secret {
		auth_request /oauth2/auth;
		error_page 401 = /oauth2/sign_in;
		auth_request_set $user $upstream_http_x_auth_request_user;
		auth_request_set $email $upstream_http_x_auth_request_email;
		proxy_set_header X-User $user;
		proxy_set_header X-Email $email;
		auth_request_set $auth_cookie $upstream_http_set_cookie;
		add_header Set-Cookie $auth_cookie;
    }
```

終わったらチェック → リロード！

```bash
sudo service nginx configtest
sudo service nginx reload
```

### 動作確認

いよいよ、認証をかけた場所にブラウザでアクセスしてみます。

{{<figure src="/img/ss/oauth2proxy2.png" class="center">}}

ボタンが出現しました！

なお、メールアドレスの制限を入れていた場合は、次のようになります。

{{<figure src="/img/ss/oauth2proxy1.png" class="center">}}

複数していだと、こんな感じ。

{{<figure src="/img/ss/oauth2proxy3.png" class="center">}}

では、ボタンを押してみましょう！

{{<figure src="/img/ss/authzero7.png" class="center">}}

Auth0 の認証画面が表示されましたか？おめでとうございます :sparkles:

もし **Callback URL mismatch.** と表示された場合は、oauth2_proxy で指定した `redirect_url` の値が Auth0 の "Settings" タブの "Allowed Callback URLs" に追加 (修正) してください。

{{<figure src="/img/ss/authzero6.png" class="center">}}

無事に認証できることまで確認したら、ミッションコンプリートです。
Auth0 のダッシュボードには、見事にログインが記録されています。

{{<figure src="/img/ss/authzero8.png" class="center">}}

GitHub みたいにアクティビティのヒートマップもありますね。まあ、Auth0 上で草を生やしてもしゃーないのですが :innocent:

### まとめ

以上で、割と簡単に Nginx サイト/アプリの OAuth 化ができることがお分かりいただけたかと思います (実は手順はほどほどにありましたが...)。
1度段取りを覚えれば、いままで BASIC や Digest だったところもどんどん OAuth 化したくなってくるかもしれません。
また、Auth0 にさえつないでおけば、こっちのサービスはこのトークン、あっちのサービスはこのトークンなどととっちらかることもなくスマートに ID を管理することができます。さらに有償版なら、エンタープライズな統合機能もたくさんあります。
こうした便利で簡単な道具をうまく組み合わせてセキュアなウェブサイトやウェブアプリをサクサク構築していきたいものですね。

Stay tuned & Happy hacking!

#### 参考文献

1. [Htpasswd Generator &#8211; Create htpasswd - Htaccess Tools](http://www.htaccesstools.com/htpasswd-generator/)
2. [oauth2_proxy/README.md at master · bitly/oauth2_proxy](https://github.com/bitly/oauth2_proxy/blob/master/README.md)
3. [Module ngx_http_auth_request_module](http://nginx.org/en/docs/http/ngx_http_auth_request_module.html)
