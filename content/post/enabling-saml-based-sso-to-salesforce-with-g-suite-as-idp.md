---
title: "G Suite を IdP にした Salesforce の SAML SSO の有効化"
date: 2017-12-07T00:19:59+09:00
draft: false
categories: ["salesforce"]
tags: [
  "gsuite",
  "saml",
  ]
---

唐突に G Suite と Salesforce を SAML で連携する方法をご紹介。

{{<figure src="/img/logo/gsuite-salesforce.png" class="center">}}

G Suite と Salesforce は共に SAML で SSO (シングルサインオン) 連携できる機能が標準で備わっており、またどちらも IdP (認証プロバイダー) を担うことができます。
今回、ひょんなことから設定を検討することになったのですが、公式ドキュメント<sup>[参考文献1]</sup>があまりにも説明足らずだったので、スクショを交えてじっくり解説したいと思います。

### 事前条件

G Suite の事前条件は以下の通りです:

- 特権管理者権限を持っている

Salesforce の事前条件は以下の通りです:

- システム管理者権限を持っている
- カスタムドメイン (xxx.my.salesforce.com) を設定してある

今回の例では、Google Apps Standard 無料時代に作成した G Suite 組織と、Salesforce Developer Edition 組織をフェデレーションします。

### 1. G Suite の情報取得

1-1. `https://admin.google.com/<DOMAIN>.com` にアクセスし、ログインします。

{{<figure src="/img/ss/gsuite-setting1.png" title="G Suite 管理コンソール" class="center" width="80%">}}

1-2. [セキュリティ] -> [シングルサインオン (SSO) の設定] と進みます。

{{<figure src="/img/ss/gsuite-setting2.png" title="シングルサインオン (SSO) の設定" class="center" width="80%">}}

1-3. [証明書をダウンロード] ボタンを押し、証明書をダウンロードしておきます。また [IDP メタデータをダウンロード] ボタンを押し、メタデータ XML もダウンロードしておきます。

1-4. このページはそのままにして別のタブで Salesforce にログインします。

### 2. Salesforce の有効化

2-1. 設定から [ID] -> [シングルサインオン設定] を選びます。

{{<figure src="/img/ss/salesforce-setting-sso1.png" title="シングルサインオン設定" class="center" width="80%">}}

2-2. [編集] ボタンを押し、[SAML を有効化] に :white_check_mark: を入れ、[保存] ボタンを押します。

2-3. [SAML シングルサインオン設定] ペインの [メタデータファイルから新規作成] ボタンを押します。メタデータファイルは手順 1-3 の IDP メタデータの XML ファイルを指定します。

{{<figure src="/img/ss/salesforce-setting-sso2.png" title="シングルサインオン設定" class="center" width="80%">}}

2-4. 以下の通り入力します (手をつけるのは**名前**と**IDプロバイダの証明書**ぐらいで、他はアップロードしたメタデータから既に入力されています):

| 入力項目                    | 入力するもの                     | 例    |
| -------------------------- | ------------------------------- | ------ |
| 名前                        | 適宜 (認証時のボタン名・設定名になる) | G Suite |
| API 参照名                  | (名前入力後に自動補完)           | G_Suite |
| 発行者                      | 手順 1-4 の [エンティティ ID] 値 | `https://accounts.google.com/o/saml2?idpid=xxx` |
| エンティティ ID             | Salesforce 組織の URL            | `https://mikan-dev-ed.my.salesforce.com` |
| ID プロバイダの証明書        | 手順 1-4 でダウンロードした証明書 | xxx.pem |
| 証明書の署名要求             | (デフォルト値) | |
| 署名要求メソッド             | RSA-SHA256 | |
| アサーション復号化証明書      | アサーション暗号化なし | |
| SAML ID 種別                | (デフォルト値) | |
| SAML ID の場所              | ID は、Subject ステートメントの NameIdentifier 要素にあります | |
| サービスプロバイダの起動要求バインド | [リダイレクト] を選択      | |
| ID プロバイダのログイン URL  | 手順 1-4 の [SSO の URL] 値      | `https://accounts.google.com/o/saml2/idp?idpid=xxx` |
| カスタムログアウト URL	      | (空) | |
| カスタムエラー URL	        | (空) | |
| シングルログアウトを有効にする | (デフォルト値) | |
| ユーザプロビジョニングの有効化 | (デフォルト値) | |

2-5. [保存] ボタンを押すと設定情報が表示されます。一番下の [エンドポイント] ペインにある情報をあとで使うため、タブをそのままにしておきます。

### 3. G Suite の IdP 設定

3-1. 管理コンソールのトップへ戻り、[アプリ] を選びます。

{{<figure src="/img/ss/gsuite-setting3.png" title="アプリの設定" class="center" width="80%">}}

3-2. [SAML アプリ] を選びます。

{{<figure src="/img/ss/gsuite-setting4.png" title="アプリ > SAML アプリ" class="center" width="80%">}}

3-3. [サービスやアプリをドメインに追加] を選択、または右下の **+** ボタン押します。

{{<figure src="/img/ss/gsuite-setting5.png" title="SAML アプリケーションで SSO を有効にする" class="center">}}

3-4. [Salesforce] を選ぶと、手順 1-4 と同じ情報が表示されるので、確認して [次へ] を選びます。

3-5. アプリケーション名に `Salesforce`、アプリケーション ID に `salesforce` が設定されています。そのまま [次へ] を選びます。

3-6. 以下の通り入力します:

| 入力項目       | 入力するもの                      | 例     |
| -------------- | ------------------------------- | ------ |
| ACS の URL     | 手順 2-5 の [ログイン URL] 値     | `https://mikan-dev-ed.my.salesforce.com?so=xxx` |
| エンティティ ID | Salesforce 組織 (ドメイン) の URL | `https://mikan-dev-ed.my.salesforce.com` |
| 開始 URL       | Salesforce 組織 (ドメイン) の URL | `https://mikan-dev-ed.my.salesforce.com` |
| 署名付き応答    | :white_large_square: (無効)      | |
| 名前 ID        | メインのメールアドレス             | |
| 名前 ID の書式  | UNSPECIFIED | |

3-6. [次へ] を選ぶと、ウィザードは完了ですが、まだ有効化されていせん。ここではひとまず [OK] を押します。

{{<figure src="/img/ss/gsuite-setting6.png" title="Salesforce の SSO 設定" class="center">}}

3-6. [Salesforce の設定] ペインの右上にあるメニューアイコンを押し、 [オン (すべてのユーザー)] を選びます。なお、組織が複数あって、個別に有効化したいときは、[一部の組織に対してオンにする] が利用できます。

{{<figure src="/img/ss/gsuite-setting7.png" title="アプリ > SAML アプリ > Salesforce の設定" class="center" width="80%">}}

### 4. Salesforce の認証有効化

4-1. 設定から [会社の設定] -> [私のドメイン] を選びます。

4-2. 一番下の [認証設定] ペインで [編集] ボタンを押します (Classic に飛びます)。

{{<figure src="/img/ss/salesforce-setting-auth1.png" title="認証設定" class="center" width="80%">}}

4-3. [認証サービス] に [G Suite] が出現しています (手順2-4 で設定した名前)。:white_check_mark: を入れて保存します。

### いざ、ログイン！

Salesforce 組織のログインページ (例では `https://mikan-dev-ed.my.salesforce.com/`) にアクセスすると・・・

{{<figure src="/img/ss/salesforce-login1.png" title="ログイン画面" class="center" width="80%">}}

"または次を使用してログイン:" が登場し、[G Suite] が出ました (手順2-4 で設定した名前)。これをクリックすると、Google の認証画面に飛ぶ仕組みです。

ログインできましたか？おめでとうございます:tada: 一本通ったら、ユーザーのプロビジョニングなどをセットアップすることもできます<sup>[参考文献2]</sup>。設定すると、G Suite でユーザーの作成や変更があった場合に、それらの情報を Salesforce に連携させることが可能になります。双方のライセンス (ユーザー数) などにも注意する必要がありますが、ある程度の規模で運用するなら業務効率化のため必須でしょう。検討してみてください。

#### 参考文献

1. [Salesforce クラウド アプリケーション - G Suite 管理者 ヘルプ](https://support.google.com/a/answer/6194938)
2. [Salesforce のユーザー プロビジョニングの設定 - G Suite 管理者 ヘルプ](https://support.google.com/a/answer/6294811)
