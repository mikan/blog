---
title: "STM32L4 Discovery Kit IoT Node に Amazon FreeRTOS を導入する"
date: 2017-12-13T23:31:44+09:00
draft: false
categories: ["iot"]
tags: [
  "aws",
  "cloudwatch",
  "freertos",
  "c++",
  ]
---

Amazon FreeRTOS は 2017/11/29 に Amazon のイベント re:Invent 2017 にて発表されたマイクロコントローラー向けリアルタイム OS のディストリビューションです。
今回、とりあえず触ってみようと対応ボードを買ってきてセットアップしてみたので、その手順をご紹介。

{{<figure src="/img/photo/20171212-2101.jpg" class="center" width="80%">}}

FreeRTOS 自体は 2003 年 Richard Barry 氏により開発されたリアルタイム OS です<sup>[参考文献1]</sup>。これまでは彼の会社である Real Time Engineers Ltd. によりメンテされていましたが、re:Invent 2017 で発表があったように今後は AWS によりメンテされることになります。また、ライセンスも例外付き GPL から MIT License に変更され、より商業利用に適したライセンスになりました。更に、ソースコードも GitHub でホスティングされるようになりました<sup>[参考文献2]</sup>。

[Amazon FreeRTOS](https://aws.amazon.com/jp/freertos/) はこの FreeRTOS (kernel) と Amazon FreeRTOS Libraries の組み合わせで構成されており、Amazon FreeRTOS Libraries には以下の3つの役割が含まれています<sup>[参考文献3]</sup>:

- MQTT と Device Shadow を用いたデバイスと AWS IoT Cloud のセキュアな接続
- AWS Greengrass コアのディスカバリと接続
- Wi-Fi 接続の管理

・・・なんだか [Mbed OS](https://www.mbed.com/en/development/mbed-os/) のコンセプトと近いものを感じますね。

FreeRTOS 自体は非常に多くのマイクロコントローラーに対応しているようなのですが、Amazon FreeRTOS として現在対応が謳われているボードは以下の 3 種類になります<sup>[参考文献2]</sup>。

- Texas Instruments (TI) - [CC3220SF-LAUNCHXL](http://www.ti.com/tool/cc3220sf-launchxl)
- STMicroelectronics (ST) - [STM32L4 Discovery kit IoT node](http://www.st.com/en/evaluation-tools/b-l475e-iot01a.html)
- NXP - [LPC54018 IoT Module](http://www.nxp.com/LPC-AWS-Module)

(Microchip Technology Incorporated の Curiosity PIC32MZ EF Development Board はカミングスーン!)

加えて、Windows 用のシミュレーターも提供されています。

今回チョイスしたのは ST のボードです (冒頭の写真)。**STM32L4 Discovery kit IoT node** と呼ばれていますが、正式名称は **B-L475E-IOT01A** で、もっと正確には B-L475E-IOT01A**2** となっています。[Mouser Electronics](https://www.mouser.jp/search/ProductDetail.aspx?r=511-B-L475E-IOT01A2) で ¥6,440 で購入しました。私の注文は UPS でアメリカから発送されました。

{{<figure src="/img/ss/ups1.png" class="center" width="50%">}}

基盤たった1枚を、ダラス・フォートワース空港、ルイビル空港、アンカレッジ空港、成田空港と4空港も経由して運んできてくれるなんてなんだか胸が熱くなりますね。
ちなみに、UPS がダイレクトに自宅に届けにきたときは不在だったのですが、電話が掛かってきて明日にしてと言ったらヤマト運輸に振替となりました。さすが世界の UPS！

この基盤の素晴らしいところは、いろんな IO やセンサーがごった煮になっているところです。載っているものを調べて図示してみました。

{{<figure src="/img/ss/stm32iot1.png" class="center" title="STM32L4 Discovery kit IoT node オモテ">}}

めちゃめちゃたくさん載っていますね！ある意味スマホよりすごいかも！全く、こんなに載ってたらハックしがいがあるってもんです。

それでは本題のセットアップをスタートしたいと想います (**前置きが長い**)。

### 事前準備

用意するものは以下の 2 つです:

- AWS アカウント (`AmazonFreeRTOSFullAccess` ポリシーがアタッチされた IAM, または root account)
- 対応ボード
- シリアルポートターミナルアプリ (screen, [Cool Term](http://freeware.the-meiers.org/), [Tera Term](https://ja.osdn.net/projects/ttssh2/) 等)
- Wi-Fi (Open/WEP/WPA/WPA2) 接続環境

なお、ドキュメントには `AWSFreeRTOSFullAccess` とありますが<sup>[参考文献5]</sup>、実際にはそんなポリシーはなく `AmazonFreeRTOSFullAccess` の間違いと思われます。

{{<figure src="/img/ss/iam1.png" class="center" width="80%">}}

### AWS IoT の設定

このセクションは FreeRTOS は全く関係ナシで、ひたすら AWS IoT の仕込みです。
まず、AWS IoT コンソールを開き「**モノ**」(Thing) の定義を行います。

> AWS IoT
>
> https://console.aws.amazon.com/iot

左側のメニューから [管理 -> モノ] を選び [モノの登録]、既に何かある場合は [作成] を選びます。

{{<figure src="/img/ss/awsiot1.png" class="center" width="80%">}}

一番上の [単一のモノを作成する] を選びます。

{{<figure src="/img/ss/awsiot2.png" class="center" width="80%">}}

モノにキュートな名前を付けてあげましょう。終わったら一番下までスクロールして [次へ] を選びます。

{{<figure src="/img/ss/awsiot3.png" class="center" width="80%">}}

一番上の [証明書の作成] を選びます。

{{<figure src="/img/ss/awsiot4.png" class="center" width="80%">}}

証明書が発行されたら、一通りダウンロードしておきましょう。そして [有効化] を押すのを忘れずに！

{{<figure src="/img/ss/awsiot5.png" class="center" width="80%">}}

本当に有効化しましたか？ (しつこい) ボタンが [無効化] に変わっていれば大丈夫です。[ポリシーのアタッチ] に進みましょう。

{{<figure src="/img/ss/awsiot6.png" class="center" width="80%">}}

おや、何もありませんね。大丈夫、あとで作ります。というわけでそのまま [モノの登録] に進みます。

{{<figure src="/img/ss/awsiot7.png" class="center" width="80%">}}

モノができました。これぞニッポンが誇るモノづくりです。

{{<figure src="/img/ss/awsiot8.png" class="center" width="80%">}}

・・・失礼。さっき飛ばしたポリシーを作りましょう。左側のメニューから [安全性] -> [ポリシー] を選びます。[ポリシーの作成]、既に何かある場合は [作成] に進んでください。

{{<figure src="/img/ss/awsiot9.png" class="center" width="80%">}}

なにやら難しそうな画面がでてきました。ここでやることは大きく 2 つあります。

1. 名前の入力
2. 以下の通りステートメントを 4 つ追加:

| アクション | リソース ARN | 効果 |
| --------- | ------------ | ------ |
| iot:Connect   | arn:aws:iot:ap-northeast-1:580599857604:client/**MQTTEcho** | ☑許可 |
| iot:Publish   | arn:aws:iot:ap-northeast-1:580599857604:topic/**freertos/demos/echo** | ☑許可 |
| iot:Subscribe | arn:aws:iot:ap-northeast-1:580599857604:topicfilter/**freertos/demos/echo** | ☑許可 |
| iot:Receive   | arn:aws:iot:ap-northeast-1:580599857604:topic/**freertos/demos/echo** | ☑許可 |

リソース ARN 欄は自動で入力されますが、あくまでテンプレで**サフィックス (replaceWithXXX 部分) の修正**が重要です。表の通り修正してください。全て追加し終わったら [作成] を選びます。

{{<figure src="/img/ss/awsiot10.png" class="center" width="80%">}}

作成し終わったら左側メニューから [安全性] -> [証明書] を選びます。いつのまにか (モノの定義をしたときに) 出来ている証明書を選び、アクションメニューから [ポリシーのアタッチ] を選びます。

{{<figure src="/img/ss/awsiot11.png" class="center" width="80%">}}

さっき作ったポリシーの名前は覚えていますか？忘れた？そんなあなたのために画面には既に先程作ったポリシーが現れています。チェックを入れて [アタッチ] を選びます。

{{<figure src="/img/ss/awsiot12.png" class="center" width="80%">}}

これで一通り仕込みは済んだはず。それでは次のステップへ行きましょう。

### Amazon FreeRTOS の開発環境構築

FreeRTOS をダウンロードしましょう。以下のリンクか、または IoT コンソールの左下の [ソフトウェア] -> [Amazon FreeRTOS デバイスソフトウェア] ペインの [ダウンロードの設定] からも行くことができます。

> AWS IoT
> 
> https://console.aws.amazon.com/freertos

今回のお目当ては「Connect to AWS IoT - ST」です。ダウンロードします。高々 2.38 MB しかありません。ダウンロードしたら、ZIP を展開しておきましょう。

{{<figure src="/img/ss/awsiot13.png" class="center" width="80%">}}

展開すると AmazonFreeRTOS ディレクトリが入っているかと思います。この中のデモを開発環境で読み込んで、設定をちょちょいといじって配備するわけですが、か、開発環境？どこにあるの？なんか `.project` とかあるけど？Eclipse か！？ (※AmazonFreeRTOS/demos/st/stm32l475_discovery/ac6)

Eclipse は半分正解で、半分間違いです。**System Workbench for STM32** という Eclipse ベースの開発環境を使います。以下のサイトからダウンロードします:

> OpenSTM32 Community Site | HomePage
>
> http://www.openstm32.org/

開くとこんな感じのウェブサイトが (ブラウザの「安全ではありません」警告と共に) 現れるはずです。

{{<figure src="/img/ss/openstm32-page1.png" class="center" width="80%">}}

悲しいことに、開発環境をダウンロードするにはログインしないといけないのです (実はファイル名がわかっていれば[ココ](http://www.ac6-tools.com/downloads/SW4STM32/)からダウンロードもできますが、一応案内上ログインしないといけないことにさせてください:innocent:)。

しかも、https ではないサイトで入力フォームに必要事項を全て記入しないといけません。しかもしかも、MS メールだと弾かれるらしいです。

{{<figure src="/img/ss/openstm32-page2.png" class="center" width="80%">}}

登録が済んだら、次の URL からようやくダウンロードの案内にありつけます。

> OpenSTM32 Community Site | Downloading the System Workbench for STM32 installer
>
> http://www.openstm32.org/Downloading+the+System+Workbench+for+STM32+installer

{{<figure src="/img/ss/openstm32-page3.png" class="center" width="80%">}}

Windows 10 64bit な私は「Windows 7」の 64bit 版をダウンロードしました。**433MB** もあり、しかもダウンロードサーバーは激しく遅いです。ダウンロードをはじめたら、お風呂にでも入りに行きましょう (私は4時間ぐらいかかりました。90年代か！不安な場合は wget などレジューム可能なツールを使ったほうが良いかもしれません)。

ダウンロードが終わったら、実行です。基本全部デフォルトで進めたほうが無難でしょう。もしエラーに遭遇したら [FAQ](http://www.openstm32.org/faq1) が助けになるかもしれません。

{{<figure src="/img/ss/swb1.png" class="center">}}

途中、インストール終盤では、ドライバをインストールする画面が別途出現します (Windows の場合)。

{{<figure src="/img/ss/swb2.png" class="center">}}

全てうまくいくと、こんな感じ。[Done] で閉じます。

{{<figure src="/img/ss/swb3.png" class="center">}}

スタートメニューやデスクトップショートカットやら作る設定 (デフォルト) なら、出来ているはずなのでそこから起動します。

{{<figure src="/img/ss/swb4.png" class="center">}}

起動スプラッシュはなかなか。起動途中に現れる Eclipse 同様のワークスペース選択設定は適当に。

{{<figure src="/img/ss/swb5.png" class="center">}}

起動後も初回インストール処理のようなものが走ります。Eclipse IDE for C/C++ Developers がベースになっていることがわかります。

{{<figure src="/img/ss/swb6.png" class="center" width="80%">}}

メニューから [File] -> [Import] を選び、Import 画面にて [General] -> [Existing Project into Workspace] を選んで [Next >] を選びます。

{{<figure src="/img/ss/swb7.png" class="center">}}

このセクションの冒頭でダウンロードした ZIP の中にある `AmazonFreeRTOS/demos/st/stm32l475_discovery/ac6` ディレクトリを指定し、`aws_demos` プロジェクトが選択されていることを確認します。最後に [Finish]!

{{<figure src="/img/ss/swb8.png" class="center">}}

Welcome タブを閉じてプロジェクトファイルを眺めたあと、メニューから [Project] -> [Build All] を選びます。

{{<figure src="/img/ss/swb9.png" class="center" width="80%">}}

:point_right:エラーよし、:point_right:ウォーニングよし、:point_right:モチベーションよし！次に進みましょう。

### IoT Node の構成

System Workbench の Project Explorer にて `aws_clientcredential.h` を開きます。`application_code/common_demos/include` にあります (ファイルシステム上のパスは `AmazonFreeRTOS/demos/common/include` です)。

{{<figure src="/img/ss/swb10.png" class="center" width="80%">}}

以下の通り値を入れていきます　(定義場所は執筆時点の最新版のものです):

| 設定値                           | 定義場所 (行) | デフォルト値               | 入力値      |
| -------------------------------- | ----------- | ------------------------- | ---------- |
| `clientcredentialMQTT_BROKER_ENDPOINT` | L39   | Paste AWS IoT Broker endpoint here. | AWS IoT のエンドポイント |
| `clientcredentialIOT_THING_NAME` | L44         | Paste AWS IoT Thing name here. | モノの名前 (この例では `stm32-001`) | 
| `clientcredentialWIFI_SSID`      | L59         | Paste WiFi SSID here.     | SSID       |
| `clientcredentialWIFI_PASSWORD`  | L64         | Paste WiFi password here. | パスワード  |
| `clientcredentialWIFI_SECURITY`  | L71         | `eWiFiSecurityWPA2`       | `eWiFiSecurityOpen`, `eWiFiSecurityWEP`, `eWiFiSecurityWPA`, `eWiFiSecurityWPA2` |

AWS IoT のエンドポイントは、[IoT コンソール](https://console.aws.amazon.com/iot)の左側メニューの下にある [設定] より確認できる値です。

{{<figure src="/img/ss/awsiot14.png" class="center" width="80%">}}

次に、Project Explorer にて `config/aws_wifi_config.h` を開きます。これも先程と同様に、以下の箇所を修正します。ちなみにこれは公式ドキュメントにはない手順なのですが、設定しないと動きませんでした。

| 設定値                                | 定義場所 (行) | デフォルト値       | 入力値      |
| ------------------------------------ | ------------- | ----------------- | ---------- |
| `wificonfigACCESS_POINT_SSID_PREFIX` | L68           | ConfigureMe       | SSID       |
| `wificonfigACCESS_POINT_PASSKEY`     | L73           | awsiotdevice      | パスワード  |
| `wificonfigACCESS_POINT_SECURITY`    | L90           | eWiFiSecurityWPA2 | `eWiFiSecurityOpen`, `eWiFiSecurityWEP`, `eWiFiSecurityWPA`, `eWiFiSecurityWPA2` |

ファイルの編集が終わったら、次は証明書の設定です。

ファイルブラウザ (エクスプローラー) にて `AmazonFreeRTOS/demos/common/devmode_key_provisioning/CertificateConfigurationTool` にある `CertificateConfigurator.html` を開きます。

{{<figure src="/img/ss/freertos1.png" class="center" width="80%">}}

それぞれ指定するものは以前ダウンロードしておいた以下のファイルです:

- Certificate PEM file: `xxxxxxxxxx-certificate.pem.crt`
- Private Key PEM file: `xxxxxxxxxx-private.pem.key`

ファイルを指定したら、[Generate and save aws_clientcredential_keys.h] を選択します。

生成されたファイルは `AmazonFreeRTOS/demos/common/include` に配置 (上書き) します。

メニューから [Project] -> [Build All] を選び、コンパイルエラーがないかチェックします。

{{<figure src="/img/ss/swb9.png" class="center" width="80%">}}

### デモ！

AWS IoT の左側のメニューより [テスト] を選び、MQTT サブスクリプションの設定を行います。トピック名 `freertos/demos/echo` とし、[MQTT ペイロード表示] は [ペイロードを文字列として表示 (より正確)] を選びます。最後に [トピックへの...] に進みます (何!? 英語は Subscribe to topic なので「トピックへのサブスクライブ」かな)。

{{<figure src="/img/ss/awsiot15.png" class="center" width="80%">}}

なお、このトピック名は `application_code/common_demos/source/aws_hello_world.c` の `echoTOPIC_NAME` 定義にあるもので、かつアタッチしたポリシーで記載したものと一致している必要があります。

AWS IoT の準備が整ったら、次は基盤のほうをチェックです。パッケージのウラにも文書で書いてあるように、以下のようなコンフィギュレーションになっていることを確認します:

| Jumper | State  |
| ------ | ------ |
| JP4    | 5V_ST_LINK |
| JP5    | Closed |
| JP6    | Closed |
| JP7    | Closed |
| JP8    | Open   |

{{<figure src="/img/ss/stm32iot2.png" class="center" title="STM32L4 Discovery kit IoT node ウラ">}}

PC とボードを USB で接続します！繋ぐポートは USB ST-LINK (端っこの穴) です。向きに注意して優しくインサートしてください:two_hearts:

{{<figure src="/img/photo/20171213-2032.jpg" class="center" width="80%">}}

System Workbench のメニューで [Run] -> [Run Configurations...] を選び、左側ペインで [Ac6 STM32 Debugging] -> [aws_demos Debug] を選びます。さらに [Debugger] タブを開き、中腹にある [Show generator options...] を押します。

{{<figure src="/img/ss/swb11.png" class="center">}}

[Connection Setup] の [Frequency] は [480MHz], [Mode Setup] の [Reset Mode] は [Software system reset] を選びます (または確認します)。設定を終えたら [Apply] ボタンで反映し、[Run] を押します。

こんな感じのログが流れるはずです。

```
Open On-Chip Debugger 0.10.0-dev-00005-g4030e1c-dirty (2017-10-24-08:00)
Licensed under GNU GPL v2
For bug reports, read
	http://openocd.org/doc/doxygen/bugs.html
none separate

(中略)

verified 306816 bytes in 9.605227s (31.194 KiB/s)
** Verified OK **
** Resetting Target **
Info : Stlink adapter speed set to 480 kHz
adapter speed: 480 kHz
shutdown command invoked
```

途中 ERROR とかも出てるけど大丈夫なんか・・・？:innocent: でも Verifyied OK とか言ってるし・・・？おそるおそる、シリアルポート端末アプリを開き (ここでは Cool Term)、Baudrate を 115200 にしてつなぎにいきます。

{{<figure src="/img/ss/coolterm3.png" class="center">}}

Cool Term は [OK] を押したあと、[Connect] を押す必要があります。繋がったら、ボードの **リセットボタン** (黒い物理ボタン) を押します。

{{<figure src="/img/ss/coolterm4.png" class="center">}}

こんな感じでログが流れていればたぶん成功しています (TONKOTSU は我が家の AP です:ramen:)。途中で止まったりしている場合はなにかがおかしいです。詳しくは後述のトラブルシューティングのセクションで解説します。

最初に開いておいた AWS IoT のトピック画面を見てみましょう！

{{<figure src="/img/ss/awsiot16.png" class="center" width="80%">}}

ぞくぞくと届く Hello World たち！いいですね！:tada:

だいぶ記事も長くなってきたので、本編はこれでおしまいにします。今後の活用もお楽しみに！

### トラブルシューティング

さて、問題に直面したあなた (っていうかわたし) はより深い知見を得る絶好のチャンスです。切り分けの基本はまず事実を集めることです。というわけでログの確認から。例によって、AWS IoT は CloudWatch によるログ集計が可能です。が、デフォルトでオフになっています。有効にするには、左側メニューの下部にある [設定] から [ログ] ペインを確認します。

{{<figure src="/img/ss/awsiot17.png" class="center" width="80%">}}

レベルは [デバッグ]、ロールは新規に作ります。名前はなんでも良いです。

{{<figure src="/img/ss/awsiot18.png" class="center" width="80%">}}

設定できたら CloudWatch のコンソールに向かいます。

> CloudWatch
>
> https://console.aws.amazon.com/cloudwatch

[メトリクスの参照] を選択します。

{{<figure src="/img/ss/cloudwatch1.png" class="center" width="80%">}}

お、[IoT] ができていますね。選択し、続けて [プロトコルメトリクス] を選択します。

{{<figure src="/img/ss/cloudwatch2.png" class="center" width="80%">}}

MQTT や HTTP などいくつかのメトリクスが並んでいるので、まずは MQTT 系を全てチェックしグラフを眺めます。期待したものがなければ AWS IoT と疎通ができていないことが推測できますし、図のように `Connect.AuthError` が頻発しているときは、疎通はできているがポリシー違反などで弾かれていると推測できます。

{{<figure src="/img/ss/cloudwatch3.png" class="center" width="80%">}}

疎通ができていない場合に関しては、どのレイヤーで止まっているかを切り分ける必要があります。幸い WiFi 接続に関してはシリアルポートに結果が出るので、問題があればすぐに特定できます。

WiFi は OK (AP や DHCP で振られた IP などが確認できる) が、その後で止まってしまった場合は解析が必要です。一番ありそうなのが、AWS IoT 通信で必須な TLS まわりでしょう。そこで、 `aws_mqtt_agent.c` の L1268 あたりに次のコードを挿入してみます。

```
configPRINTF( ( "DEBUG: prvManageConnections() lBytesReceived=%d\r\n", lBytesReceived) );
```

ログはこうなりました。

```
14 28165 [MQTT] DEBUG: prvManageConnections() lBytesReceived=-1004
15 32421 [MQTT] DEBUG: prvManageConnections() lBytesReceived=-1004
16 33425 [MQTT] DEBUG: prvManageConnections() lBytesReceived=-1
17 34428 [MQTT] DEBUG: prvManageConnections() lBytesReceived=-1
...
```

数値の値は定義済のものです。`aws_secure_sockets.h` では以下のように定義されているのが確認できます:

- `0`: 成功
- `-1`: ソケットエラー
- `-11`: 資源が一時的に利用できない
- `-12`: メモリアロケーション失敗
- `-22`:` 不正な引数
- `-109`: 不正なオプション指定
- `-128`: 提供されたソケットが既にクローズされている
- `-1001`: TLS 初期化失敗
- `-1002`: TLS ハンドシェイク失敗
- `-1003`: 接続は確率したがサーバーを検証できない、ソケットのクローズがおすすめ
- `-1004`: TLS 受信処理失敗
- `-1005`: TLS 送信処理失敗
- `-1006`: 通信機器がリセットされた

上のログでは、どうやらデータ受信に失敗していたようです。私の場合はこのあと直っちゃったのですが、TLS 実装部まで追う場合は `aws_tls.c` をデバッグしていくことになります。ちなみにこのコード、実は Mbed OS などで使われている [mbed TLS](https://tls.mbed.org/) なんです！んじゃ諦めて Mbed OS に乗りｋ・・・いやいや、もう少し頑張るんだ・・・:innocent: なお、先程用いたシリアルコンソール表示 `configPRINTF` はここでは `#include "FreeRTOS.h"` しないと使えません。

疎通はできていて、CloudWatch で `Connect.AuthError` が記録される場合は、使った証明書が (証明書としては正しいけれど) 別のものだったか、証明書に適切なポリシーがない、ないしは間違っている可能性があります。過去の手順を今一度確認してみてください。

Happy hacking!

#### 参考文献

1. [RTOS - Free professionally developed and robust real time operating system for small embedded systems development](https://www.freertos.org/RTOS.html)
2. [aws/amazon-freertos: Cloud-native IoT operating system for microcontrollers.](https://github.com/aws/amazon-freertos)
3. [What Is Amazon FreeRTOS? - Amazon FreeRTOS](https://docs.aws.amazon.com/freertos/latest/userguide/what-is-amazon-freertos.html)
4. [アナウンス Amazon FreeRTOS – 何十億ものデバイスがクラウドから安全に利益を得ることを可能にする | Amazon Web Services ブログ](https://aws.amazon.com/jp/blogs/news/announcing-amazon-freertos/)
5. [Amazon FreeRTOS - Get Started - AWS](https://aws.amazon.com/jp/freertos/getting-started/)
6. [B-L475E-IOT01A - STM32L4 Discovery kit IoT node,  low-power wireless, BLE, NFC, SubGHz, Wi-Fi - STMicroelectronics](http://www.st.com/ja/evaluation-tools/b-l475e-iot01a.html)
7. [Getting starting with STM32L4 Discovery kit IoT node - YouTube](https://www.youtube.com/watch?v=6eUqxjBL_wI)
8. [Amazon FreeRTOS Security - Amazon FreeRTOS](http://docs.aws.amazon.com/ja_jp/freertos/latest/userguide/freertos-security.html)
