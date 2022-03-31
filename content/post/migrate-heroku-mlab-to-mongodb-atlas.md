---
title: "Heroku の mLab Add-on を MongoDB Atlas に移行する"
date: 2020-07-11T11:56:54+09:00
draft: false
categories: ["web"]
tags: [
  "heroku",
  "java",
  "spring-boot",
  "mongodb"
]
---

{{<figure src="/img/logo/mlab_rip.png" class="center" width="50%">}}

先日 (2020/7/10), Heroku のアドオン mLab MongoDB が 2020/11/10 にシャットダウンするとアナウンスがありました<sup>[参考文献1]</sup>。

mLab (旧 MongoLab) といえば、Heroku 界隈では無料でサクッと MongoDB を試せる便利な DBaaS として多くの採用例がありました。
有償プランも充実しており、最大ストレージ 700GB / RAM 64GB までスケールさせることができるのも強みで、本番用には有償プラン、開発用環境や Review Apps などでは無料の sandbox、みたいに使っていた方も多いのではないでしょうか。

しかし 2018/10/9 に MongoDB の開発元 MongoDB, Inc. が mLab を買収します<sup>[参考文献2]</sup>。
MongoDB, Inc. では独自の DBaaS である Atlas をローンチしており、mLab は完全に競合サービスとなっていました。買収アナウンスによれば今後 12 ヶ月以内にすべての顧客を Atlas に移行させることを期待しているとあり、競合潰しのための買収であることは明らかです。
この買収と方針を受け、mLab ではマイグレーションを支援するためのツールを開発する方針を示し<sup>[参考文献3]</sup>、実際にリリースされました。しかしマイグレーションガイドは 11 ステップもあり、ワンクリックで終わり！みたいなツールではありません。

### 移行先の選定

では、残された時間で開発者が取れる選択肢はどれだけあるでしょうか？

Heroku はシャットダウン予告のメールの中で別の MongoDB Add-on の「ObjectRocket」への移行を提案しています。

{{<figure src="/img/ss/heroku2.png" class="center" width="50%">}}

Heroku Add-on の使い勝手の良さを引き続き享受するにはこれしかありません。引き続き Add-on 代を徴収したい Heroku としても、こちらを推したいはずです (※ Add-on 自体の提供元は Heroku ではないサードパーティ)。

とはいえマイグレーションの手間がかかることには代わりありません。それに ObjectRocket にはたくさんの Heroku ユーザーが求めている無料 Sandbox がありません！ 
Atlas には無料 Sandbox があります。やっぱり Atlas しかありませんね！

なお、ObjectRocket や Atlas 以外にもまだまだ選択肢もあります。一応紹介しておきます。

- Azure Cosmos DB (API 互換のベツモノ)
- Amazon DocumentDB (API 互換のベツモノ)
- IBM Cloud Hyper Protect DBaaS for MongoDB (ホンモノの MongoDB クラスター)

DBaaS 自体をやめるという選択肢もあります。でも MongoDB を自分でホストするのは大変な苦痛を伴います。個人的にはおすすめできません。

本記事では、表題どおり MongoDB Atlas を選定します。11 ステップの地獄のマイグレーション作業の始まりです。コーヒーと頭痛薬を用意したら、早速はじめましょう！

なお本記事は mLab の Sandbox (無料) を Atlas の Sandbox (無料) に移行することを想定しています。有償の場合は手順が異なる場合があります。

### マイグレーション前の準備

#### A. Atlas 組織とプロジェクトの作成

次の URL にアクセスしてサインアップします。

> Managed MongoDB Hosting | Database-as-a-Service | MongoDB
>
> https://www.mongodb.com/cloud/atlas

{{<figure src="/img/ss/atlas1.png" class="center" width="80%">}}

Start free ボタンを押して、個人情報を入力してサインアップします。

私は実行したら `CORS_ORIGIN_DENIED` というエラー画面になってしまいました😇
先行きがとっても不安です・・・。しかし大丈夫、ちゃんとサインアップメールが届いていました。メールのリンクをクリックしてログインします。

{{<figure src="/img/ss/atlas2.png" class="center" width="80%">}}

同じ画面になりましたか？それでは次のステップです。

#### B. Atlas 組織と　mLab アカウントの接続

{{<figure src="/img/ss/atlas3.png" class="center">}}

画面左上の葉っぱ (失礼) の右側にある歯車アイコン⚙️をクリックし、 "Organization Settings" 画面を開いてください。

{{<figure src="/img/ss/atlas4.png" class="center" width="80%">}}

一番下までぐぐっとスクロールすると、"Connect to mLab" というペインが登場しています！素晴らしい！
早速 [Connect to mLab] ボタンを押します。

{{<figure src="/img/ss/mlab1.png" class="center" width="80%">}}

mLab のログイン画面になりますね。でも Heroku Add-on だとアイパスがわかりません。
で、どうするかというと、ここで一旦、 **同じブラウザで別タブ** を開き、Heroku Dashboard にアクセスします。

> Heroku Dashboard
>
> http://dashboard.heroku.com

ログイン後、目的のアプリを開きます。そして "Installed add-ons" ペインにある mLab をクリックし、mLab のダッシュボードを開きます。

{{<figure src="/img/ss/heroku3.png" class="center">}}

mLab のダッシュボードが開きましたね！この時点でブラウザには次の 3 つのタブがあるはずです。

1. mLab ログイン画面のタブ
2. Heroku Dashboard のタブ
3. mLab のタブ

タブ 3 から 1 に切り替え、 **ログイン画面をリロード** します。

するとあら不思議、画面が変わりました。

{{<figure src="/img/ss/mlab2.png" class="center" width="80%">}}

次は **ブラウザの戻るボタン**で Atlas に戻ります。

{{<figure src="/img/ss/atlas4.png" class="center" width="80%">}}

もう一回 [Connect to mLab] ボタンを押します。
あらあら不思議、新しい画面が現れました。

{{<figure src="/img/ss/mlab3.png" class="center" width="80%">}}

[Authorize] ボタンを押すと、Atlas の画面に戻ります。

{{<figure src="/img/ss/atlas5.png" class="center" width="80%">}}

### 指定したデータベースのマイグレーション

#### C. 指定したデータベースのマイグレーション処理の開始

先程の画面で行の右端に [...] ボタンがあります。押してメニューを開き、続いて [Configure Migration] を選択します。

{{<figure src="/img/ss/atlas6.png" class="center">}}

お待たせしました・・・こちらが「マイグレーションウィザード」です！

冒頭で言及した「11ステップ」というのはここから始まります！ (いままでのは事前準備...)

{{<figure src="/img/ss/atlas7.png" class="center" width="80%">}}

#### D. マイグレーションウィザードのタスクの実施

**Step 1. 対象プロジェクトの設定**

マイグレーションウィザードの1番目のボタン [Create or Select Target Project] を押しましょう。

{{<figure src="/img/ss/atlas8.png" class="center" width="80%">}}

"Project 0" がありましたのでこれを選択します。いったいつ作ったんでしょう？きっとサインアップしたときですね。もしなければ画面に従って作ってください。

準備できたら [Confirm Project and Continue] ボタンを押します。

**Step 2. データベースユーザーのインポート**

{{<figure src="/img/ss/atlas9.png" class="center" width="80%">}}

マイグレーションガイドでは特に何も指示されていません。警告アイコンが気になりますが、そのまま進めましょう。
[Import Database Users And Continue] ボタンを押します。

**Step 3. IP ホワイトリストの設定**

{{<figure src="/img/ss/atlas10.png" class="center" width="80%">}}

"Allow all IP address..." と続くチェックボックスを入れます。Heroku は原則ソース IP が不定ですからね。

チェックを入れたら [Allow All And Continue] ボタンを押します。

**Step 4. 対象 Atlas クラスターの作成**

{{<figure src="/img/ss/atlas11.png" class="center" width="80%">}}

"Cluster" のプルダウンを開くと "Create most equivalent new cluster RECOMMENDED" という唯一の選択肢があるので選択します。

するとクラスターの設定を選ぶ重要な画面になります。各項目を確認しながら一番下までスクロールしてください。

{{<figure src="/img/ss/atlas12.png" class="center" width="80%">}}

M0 が無料の Sandbox です！お値段の見積もりも表示されていますね。M0 なら $0 なのを確認してください。
十分に確認したら [Confirm Target And Continue] ボタンを押します。

すると、クラスター作成中表示が現れます。しばらく待ちましょう。私は3分待ちました。

{{<figure src="/img/ss/atlas13.png" class="center" width="80%">}}

終わるとこんな表示になります。

{{<figure src="/img/ss/atlas14.png" class="center" width="80%">}}

**Step 5. 移行元と移行対象を確認**

マイグレーションウィザードの [Confirm Source and Target] ボタンを押します。

(and だったり And だったりが気になりますね...)

{{<figure src="/img/ss/atlas15.png" class="center" width="80%">}}

確認したら [Confirm And Continue] ボタンを押します。

**Step 6. マイグレーションのテスト**

{{<figure src="/img/ss/atlas16.png" class="center" width="80%">}}

マイグレーションのテストを実行するかどうかを選択できます。

このテストは実際に以下の手順を行い、マイグレーション実行時間の見積もりやデータ・インデックスが実際に移行可能であることを検証できます。
マイグレーションガイドでも "strongly recommend" (強く推奨) とあるので、実施しない手はないでしょう。

1. Atlas クラスターを消去
2. mLab で `mongodump` を実行
3. Atlas クラスターで `mongorestore` を実行

テストする場合はそのまま (選択肢にチェックを入れずに) [Begin Test Run] ボタンを押します。

テスト実行中は以下のような画面になります。しばらく待ちましょう。私は2分待ちました。

{{<figure src="/img/ss/atlas17.png" class="center" width="80%">}}

終わったら次のような画面になるので、 [Confirm Connectivity] ボタンを押します。

{{<figure src="/img/ss/atlas18.png" class="center" width="80%">}}

**Step 7. 接続テスト**

{{<figure src="/img/ss/atlas19.png" class="center" width="80%">}}

接続テストの案内画面になりました。

テスト方法は色々とあります。
まず、MongoDB Atlas のクラスターに接続するには 2 つの接続文字列があります<sup>[参考文献5,6]</sup>。

1. DNS Seedlist Connection Format (SRV): `mongodb+srv://<username>:<password>@<host>/<db>?retryWrites=true&w=majority`
2. Standard Connection String Format: `mongodb://<username>:<password>@<shard0_host>:27017,<shard1_host>:27017,<shard2_host>:27017/<db>?ssl=true&replicaSet=<shard0_name>&authSource=admin&retryWrites=true&w=majority`

1番目のやり方はページに書いてあります。2番目のやり方は "Having trouble connecting?" の中に書いてあります。
お勧めは当然シンプルでモダンな1番目のほうです。しかし1番目でうまくいかない場合は2番目を試すことになります。

このような仕様になっているため、単に Mongo Shell から繋いで試すだけでなく、実際のアプリケーション (ドライバー) の DB 設定を書き換えてローカルで実行する等して検証する事が重要です。

私のアプリは Java 11 + Spring Boot + Spring Data MongoDB という環境でしたが、 `spring.data.mongodb.uri` に挿して試したところ以下のようなエラーになってしまいました:

```json
{"ok": 0, "errmsg": "no SNI name sent, make sure using a MongoDB 3.4+ driver/shell.", "code": 8000, "codeName": "AtlasError"}
```

これは使っていた AdoptOpenJDK 11 が SNI をサポートしていないことによる問題で<sup>[参考文献7]</sup>、結局 [Azul Zulu OpenJDK](https://www.azul.com/downloads/zulu-community/) に置き換えて解決しました。

確認したら [Confirm And Continue] ボタンを押します。

**Step 8. Heroku アドオンの config var からの独立性確保**

{{<figure src="/img/ss/atlas20.png" class="center" width="80%">}}

マイグレーション完了後、mLab アドオンは削除することになりますが、その際に `MONGODB_URI` config var (環境変数) も Heroku App から削除されてしまいます。
そのため、`MONGODB_URI` 以外の config var を別に作り、アプリが参照する環境変数をそちらに切り替える作業を行ってください。
もちろん、複数アタッチされている場合は色の名前で区別されたそれぞれにおいて対応が必要です。

私は画面にある例そのまま `DB_URI` にすることにしました。具体的な手順は次の通り:

1. Heroku Dashboard の App の "Settings" タブを開き、　[Reveal Config Vars] ボタンを押す
2. 追加欄で KEY = `DB_URI`, VALUE = (`MONGODB_URI` の値をコピペ) を記入し [Add] ボタンを押す
3. アプリケーションの設定で参照環境変数を変更してデプロイ

{{<figure src="/img/ss/heroku4.png" class="center">}}

完了したら [Confirm And Continue] ボタンを押します。

**Step 9. マイグレーション**

いよいよマイグレーション実行のときが来ました。

{{<figure src="/img/ss/atlas21.png" class="center" width="80%">}}

必要に応じてアプリケーションを止め (メンテナンスモードにするなど)、注意事項を確認したら、 "I understand that..." のチェックボックスを入れて、

いざ、

[Begin Migrasion] を押します！

マイグレーション実行中は以下のような画面になります。しばらく待ちましょう。私は3分待ちました。

{{<figure src="/img/ss/atlas22.png" class="center" width="80%">}}

終わりましたか？おめでとうございます！では [Start Using Atlas] ボタンを押しましょう。

{{<figure src="/img/ss/atlas23.png" class="center" width="80%">}}

**Step 10. Atlas の利用を開始**

{{<figure src="/img/ss/atlas24.png" class="center" width="80%">}}

先程作った Heroku config var の `DB_URI` を、Step 7 でテストした接続文字列に置き換えます。

{{<figure src="/img/ss/heroku5.png" class="center" width="80%">}}

config ver を変更するとほぼ即座に Dyno がリスタートすることにご注意ください。

無事に変わりましたか？おめでとうございます🎉

マイグレーションウィザードはもう用無しですが、一応画面は続いているので進めましょう。

最後のステップは mLab の削除です。

**Step 11. mLab Add-on の削除**

{{<figure src="/img/ss/atlas25.png" class="center" width="80%">}}

注意事項が表示されるだけです。削除操作は自分で操作する必要があります。

Heroku Dashboard の App の "Configure Add-ons" リンクから削除できます。

{{<figure src="/img/ss/heroku6.png" class="center">}}

くちばしみたいなアイコンに "Delete Add-on" があります。

{{<figure src="/img/ss/heroku7.png" class="center">}}

削除後もアプリが動いていることを確認し、残ったコーヒーをすべて飲み干したら、全てが終了です。お疲れさまでした。

Happy hacking!

#### 参考文献

1. [Shutdown of mLab's Heroku Add-on on November 10, 2020 | mLab Documentation & Support](https://docs.mlab.com/shutdown-of-heroku-add-on/)
2. [mLab is becoming a part of MongoDB, Inc.](https://blog.mlab.com/2018/10/mlab-is-becoming-a-part-of-mongodb-inc/)
3. [Migrating to MongoDB Atlas | mLab Documentation & Support](https://docs.mlab.com/mlab-to-atlas/)
4. [Guide to Migrating a Sandbox Heroku Add-on to Atlas | mLab Documentation & Support](https://docs.mlab.com/how-to-migrate-sandbox-heroku-addons-to-atlas/)
5. [Troubleshooting connection issues to MongoDB Atlas | mLab Documentation & Support](https://docs.mlab.com/troubleshooting-atlas-connection-issues/)
6. [Connection String URI Format &mdash; MongoDB Manual](https://docs.mongodb.com/manual/reference/connection-string/)
7. [java - MongoCommandException: Command failed with error 8000 (AtlasError): &#39;no SNI name sent, make sure using a MongoDB 3.4+ driver/shell.&#39; - Stack Overflow](https://stackoverflow.com/questions/59112124/)
