---
title: "Kotlin で始める AWS Greengrass"
date: 2018-06-23T19:57:02+09:00
draft: false
categories: ["iot"]
tags: [
  "kotlin",
  "raspberrypi",
  "aws",
  "gradle",
  "intellij",
  ]
---

{{<figure src="/img/photo/20180617-1643.jpg" class="center" width="50%">}}

AWS Greengrass という、IoT 機器のローカルで AWS Lambda の関数を実行できて、他にもあんなことやこんなことができる素晴らしいサービスをご存知でしょうか？
Greengrass があれば、エッジコンピューティングで行いたいプログラムや機械学習モデルなどを効率的かつセキュアにデプロイし管理することができます。サービスは AWS IoT と統合されており、[以前紹介した Amazon FreeRTOS](/2017/12/13/setup-amazon-freertos-on-stm32l4-discovery-kit-iot-node/) などとも一体的に管理することができます。あまりの簡単さに草が生えそうです。まさにグリーン・グラス。

さて、Greengrass は Lambda を実行できると紹介しました。Lambda 自体は[以前紹介したように](/2018/01/23/data-visualization-on-static-sites-with-aws-lambda-go-and-chart-js/) Node.js, Java, Python, C#, そして Go に対応しているのですが、Greengrass で実行できるのは現在のところ Python (2.7), Node.JS (6.10), Java (8) の3種類だけとなっています<sup>[参考文献1]</sup>。
この3種類の中に好みの言語がなかったらどうしましょう？でも大丈夫、Java をサポートしているのは何よりもの救いです。
好きな JVM 言語を選べばよいのです！

今回 Greengrass Java を試すにあたって Kotlin をチョイスしてみました。
Kotlin は Android の正式な開発言語として採用され、Spring Framework にも公式サポートが加わり、Alt Java 本命としてどんどん採用が広がっています (Alt JS としても使えます！)。
そして Java ではなく Kotlin をあえて採用するモチベーションとして、ネイティブコンパイル機能 **Kotlin/Native** への期待もあります。もっとも、現状では Java 製ライブラリに頼り切った開発が予想されるため移行は容易ではなさそうですが、 Java を選ぶよりかは夢は広がります。

それでは行ってみましょう！ (**前置きが長い**)

### 対応機器の準備

Greengrass は IoT 向けとは謳われていますが、CPU 要件が 1GHz 以上 (ARM, x86), メモリ要件が空き 128MB 以上と IoT 機器としてはかなり高い要求スペックです。そして参考文献1に並ぶ認証機器はほとんどが Intel CPU を搭載したゲートウェイ機器となっており、とてもセンサーとつないでばらまくような類の機器ではありません。みんな大好き Raspberry Pi に関しては、ハイエンド機の [Raspberry Pi 3 Model B](https://amzn.to/2yixLqH) のみがサポート対象となっています。というわけで、今回はこちらを採用したいと思います。
なお、冒頭の写真は高い放熱性能を誇るアルミ合金製ケース [cocopar® Raspberry pi metal-case](https://amzn.to/2leao8k) を装着しています。Raspberry Pi 3 はアツアツになるので予めこういったケースも用意しておくと安心です。
さらに、Raspberry Pi 3 は大飯食らいなので、3A 以上の MicroUSB AC アダプターが必要です。私のチョイスはフル負荷検証済を謳う [Physical Computing Lab Raspberry Pi 用電源セット](https://amzn.to/2yfLI91)ですが、コイル鳴きがうるさいのが難点・・・。でも、ケーブルに電源スイッチがあるのは気に入っています。頻繁に抜き差しすると、コネクタを痛めますからね。

Raspberry Pi の OS は、公式の開発者ガイドによると Raspbian Jessie、2017-03-02 となっていますが (GGC v1.5.0) <sup>[参考文献3]</sup>、今回は記事執筆時点の最新版である Raspbian Stretch Lite の April 2018 でトライしてみます。サポート対象外なのでお気をつけください。

> Download Raspbian for Raspberry Pi
>
> https://www.raspberrypi.org/downloads/raspbian/

OS 別の SD 書き込み手順は、上記サイトほか様々なところで紹介がされているので割愛します。

キーボードとディスプレイをつないで Raspbery Pi 3 に電源を供給、 Raspbian が起動したら、 ユーザー `pi` パスワード `raspberry` でログインします。その後 `sudo raspi-config` と打ち、以下の 3 点を設定します。

- **1. Change User Password** から `pi` ユーザーのパスワードを変更する
- **2. Network Options** から無線 LAN の設定を行う
- **5. Interfacing Options** から SSH サーバーの有効化を行う

終わったら `sudo reboot` で再起動し、 `pi` と新しいパスワードでログインしたら `hostname -I` で IP を確認します (あるいは起動メッセージに既に表示されているかもしれません)。

IP を確認し終えたら、以後は PC (macOS のターミナルや、Windows の WSL 等) から SSH で作業しましょう。以下は IP が 192.168.1.28 の例です。

```console
$ ssh pi@192.168.1.28
The authenticity of host '192.168.1.28 (192.168.1.28)' can't be established.
ECDSA key fingerprint is SHA256:HfJul9JCZI0rnDwemqmjofHvF3H8/DlY4cdQEg3UMCo.
Are you sure you want to continue connecting (yes/no)?
```

フィンガープリントがうんちゃらと聞かれたら **yes** と入力します。`pi@raspberrypi:~ $ ` というプロンプトが出れば成功です。

無事 SSH で繋がったら、仕込みを行います。まずはあとで入れる GGC (Greengrass Core ) ソフトウェアのためのユーザー・グループの作成から。

```bash
sudo adduser --system ggc_user
sudo addgroup --system ggc_group
```

設定ファイルをいじります。

```console
sudo vi /etc/sysctl.d/98-rpi.conf
```

以下の 2 行を追加します。

```
fs.protected_hardlinks = 1
fs.protected_symlinks = 1
```

もう一つ設定ファイルをいじります。これは Raspbian Stretch を選んだ場合のみ起きる減少に対処するためです <sup>[参考文献5]</sup>。

```console
sudo vi /boot/cmdline.txt
```

行末に次を追記します (半角スペースを入れてから入力を続けてください)。

```
 cgroup_enable=memory
```


`sudo reboot` で再起動したのち再び SSH ログインして、先ほどの設定が反映されているか確認します。

```console
$ sudo sysctl -a 2> /dev/null | grep fs.protected
fs.protected_hardlinks = 1
fs.protected_symlinks = 1
```

Java 8 を導入します。ファイルサイズが大きい (約 62 MB) ので注意してください。

```bash
sudo apt update
sudo apt install oracle-java8-jdk
```

導入されたら、Greengrass が Java 8 を認識するために `java8` コマンドを作ります。
SDK の README には "Make sure the file is not a symlink." と書いてあるのですが・・・一旦無視します:sunglasses:

```bash
$ sudo ln -s /usr/bin/java /usr/bin/java8
$ java8 -version
java version "1.8.0_65"
Java(TM) SE Runtime Environment (build 1.8.0_65-b17)
Java HotSpot(TM) Client VM (build 25.65-b01, mixed mode)
```

終わったら、いったん `exit` で SSH を抜け、クラウド側の設定に移ります。

### Greengrass の準備

本手順は、AWS IoT や Greengrass が利用可能な AWS ユーザー (root/IAM) を既に持っていることを前提に進めます。

最初に準備するのは Greengrass グループです。AWS IoT コンソールにアクセスし、Greengrass -> グループを選びます。

> AWS IoT
>
> https://console.aws.amazon.com/iot/

:bulb: 画面右上に現在のリージョンが表示されています。希望するリージョンを確認・選択してから作業を開始してください。

{{<figure src="/img/ss/greengrass1.png" class="center" width="80%">}}

[グループの作成] ボタンを押すと、ウィザードが始まります。ここでは [簡単な作成の使用] を選びます (すごい直訳感・・・)。

{{<figure src="/img/ss/greengrass2.png" class="center" width="80%">}}

適当なグループ名をつけて、 [次へ] を選びます。ここでは rpi3-group としました。

{{<figure src="/img/ss/greengrass3.png" class="center" width="80%">}}

Core 機能の名前を付ける画面ですが、既にグループ名 + "_Core" が入っています。そのまま [次へ] を選びます。

{{<figure src="/img/ss/greengrass4.png" class="center" width="80%">}}

なんだかいろいろしてくれるらしいです。確認したら [グループと Core の作成] を選びます。

{{<figure src="/img/ss/greengrass5.png" class="center" width="80%">}}

GGC のリソースがダウンロードできる画面になりました。以下の 2 点をダウンロードします。

- [これらのリソースは tar.gz としてダウンロードしてください]
- ARMv7l, Raspbian Jessie, Linux の行の [ダウンロード]

{{<figure src="/img/ss/greengrass6.png" class="center" width="80%">}}

### Greengrass Core ソフトウェアの導入

先の画面でダウンロードしたファイルは、それぞれ以下の名前になっているはずです。

- _&lt;GUID&gt;_-setup.tar.gz
- greengrass-linux-armv7l-1.5.0.tar.gz

これを先ほどの Raspberry Pi 3 に SCP で転送します (_&lt;GUID&gt;_ 部と IP アドレスは一例ですので置き換えてください)。

```console
$ scp ~/Downloads/d708e6ce5d-setup.tar.gz pi@192.168.1.28:/home/pi
$ scp ~/Downloads/greengrass-linux-armv7l-1.5.0.tar.gz pi@192.168.1.28:/home/pi
```

パスワードを聞かれたら、 Raspberry Pi 3 の `pi` ユーザーに設定したものを入力します。

転送が終わったら、SSH でログインし直します。

```
$ ssh pi@192.168.1.28
```

ファイルを確認します (_&lt;GUID&gt;_ 部は...以下略)。

```console
$ cd
$ ls -lh
total 8.2M
-rw-r--r-- 1 pi pi 2.7K Jun 17 10:24 d708e6ce5d-setup.tar.gz
-rw-r--r-- 1 pi pi 8.2M Jun 17 10:24 greengrass-linux-armv7l-1.5.0.tar.gz
```

展開します (_&lt;G_ ...以下略)。

```console
$ sudo tar -xzvf greengrass-linux-armv7l-1.5.0.tar.gz -C /
$ sudo tar -xzvf d708e6ce5d-setup.tar.gz -C /greengrass
```

ルート CA 証明書をデバイスにインストールします。

```bash
cd /greengrass/certs/
sudo wget -O root.ca.pem http://www.symantec.com/content/en/us/enterprise/verisign/roots/VeriSign-Class%203-Public-Primary-Certification-Authority-G5.pem
```

空でないか確認します。

```console
$ cat root.ca.pem
-----BEGIN CERTIFICATE-----
... (大量の文字の羅列)
-----END CERTIFICATE-----
```

:bulb: もし空の場合は、URL を確認してくださいとのことです。

次のコマンドで、 GGC を起動します。

```console
$ cd /greengrass/ggc/core
$ sudo ./greengrassd start
Setting up greengrass daemon
Validating hardlink/softlink protection
Validating execution environment
Found cgroup subsystem: cpuset
Found cgroup subsystem: cpu
Found cgroup subsystem: cpuacct
Found cgroup subsystem: blkio
Found cgroup subsystem: devices
Found cgroup subsystem: freezer
Found cgroup subsystem: net_cls

Starting greengrass daemon
Greengrass successfully started with PID: 4633
```

PID が表示されるので、生きているか確認します。

```
$ ps aux | grep 4633
root      4633  0.2  1.4 931084 13448 pts/0    Sl   11:42   0:00 /greengrass/ggc/packages/1.5.0/bin/daemon -core-dir=/greengrass/ggc/packages/1.5.0 -greengrassdPid=4606
pi        4755  0.0  0.0   4372   540 pts/0    S+   11:49   0:00 grep --color=auto 4633
```

(GGC プロセスと、今叩いている `grep` コマンド自身がひっかかります)

### Lambda 関数の作成

だいぶ仕込みに時間がかかってしまいましたが、いよいよ Lambda の実装です。

Kotlin の開発環境として、**IntelliJ IDEA** (いんてりじぇい あいでぃあ) を用意します。
Ultimate と Community がありますが、今回はどちらでも開発できます。

> Download IntelliJ IDEA: The Java IDE for Professional Developers by JetBrains
>
> https://www.jetbrains.com/idea/download/

導入時、色々なオプション機能選択ができますが、 Gradle が使えれば良いです。
インストールして立ち上がったら、 [Create New Project] を選択し、次のように [Gradle], [Kotlin (Java)] 指定して [Next] を選択します。

{{<figure src="/img/ss/idea1.png" class="center" width="80%">}}

GroupId, ArtifactId は適当に。このあとは全部デフォルトのまま [Next], [Next] と進み最後に [Finish] です。

{{<figure src="/img/ss/idea2.png" class="center" width="80%">}}

プロジェクトが開きました。

{{<figure src="/img/ss/idea3.png" class="center" width="80%">}}

`build.gradle` を開き、 `dependencies` ブロックに以下の 4 行を付け足します。

```groovy
    compile 'com.fasterxml.jackson.core:jackson-core:2.8.8'
    compile 'com.fasterxml.jackson.core:jackson-databind:2.8.8'
    compile 'org.apache.httpcomponents:httpclient:4.5.3'
    compile 'com.amazonaws:aws-lambda-java-core:1.1.0'
    compile fileTree(dir: 'libs', include: ['*.jar'])
```

また、末尾に以下を追記します。

```groovy
task buildZip(type: Zip) {
    from compileJava
    from compileKotlin
    from processResources
    into('lib') {
        from configurations.runtime
    }
}

build.dependsOn buildZip
```

編集が終わったら、 Gradle プロジェクトをリフレッシュしましょう。
画面右手にある [Gradle] というツールウインドウを開き (ツールウインドウがなければ、画面の一番左下の ❏ ←こんなアイコンをクリック)、次に再読込ボタン (下の画像だと左手にある青くて丸いボタン :cyclone:) をクリックします。

{{<figure src="/img/ss/idea4.png" class="center">}}

依存パッケージのダウンロードなどを行うため、完了までにしばらく時間がかかります。終了すると、この欄に Source Sets や Tasks がツリー上で表示されます。

次に、ディレクトリを掘っていきます。
プロジェクトを右クリックし、 [New] -> [Directory] を開きます。
画面に `src/main/kotlin/demo` と入力し、 [OK] を選びます。

{{<figure src="/img/ss/idea5.png" class="center">}}

パッケージ `demo` を右クリックし、 [New] -> [Kotlin File/Class] を選びます。
画面の Name: 欄に `HelloWorld` を入力し、 Kind: は Class を選んで [OK] を選びます。

{{<figure src="/img/ss/idea6.png" class="center">}}

`HelloWorld.kt` には以下のように記述します。

```kotlin
package demo

import java.nio.ByteBuffer
import java.util.Timer
import java.util.TimerTask

import com.amazonaws.services.lambda.runtime.Context
import com.amazonaws.greengrass.javasdk.IotDataClient
import com.amazonaws.greengrass.javasdk.model.*

class HelloWorld {
    fun handleRequest(input: Any, context: Context): String {
        return "Hello from Kotlin"
    }

    companion object {
      init {
          val timer = Timer()
          // Repeat publishing a message every 5 seconds
          timer.scheduleAtFixedRate(PublishHelloWorld(), 0, 5000)
      }
  }
}

private class PublishHelloWorld : TimerTask() {
    private val iotDataClient = IotDataClient()
    private val publishMessage = String.format("Hello world! Sent from Greengrass Core running on platform: %s-%s using Java", System.getProperty("os.name"), System.getProperty("os.version"))
    private val publishRequest = PublishRequest().withTopic("hello/world").withPayload(ByteBuffer.wrap(String.format("{\"message\":\"%s\"}", publishMessage).toByteArray()))

    override fun run() {
        try {
            iotDataClient.publish(publishRequest)
        } catch (ex: Exception) {
            System.err.println(ex)
        }
    }
}
```

さて、コードを打つと IDE は `com.amazonaws.greengrass` がないと怒ってきます。一体これはどこにあるのでしょうか。
応えは Greengrass のコンソールにあります。

コンソールに戻り、画面右下の [ソフトウェア] を開きます。スクロールして AWS Greengrass Core SDK を探し、 [ダウンロードの設定] を選びます。

{{<figure src="/img/ss/greengrass7.png" class="center" width="80%">}}

SDK ダウンロード画面になるので、プルダウンから [Java 8 バージョン 1.1.0] を選び、右の [Greengrass Core SDK のダウンロード]を選びます。

{{<figure src="/img/ss/greengrass8.png" class="center" width="80%">}}

ダウンロードしたら、展開します。

```console
$ tar zxvf aws-greengrass-core-sdk-java-1.1.0.tar.gz
$ cd aws_greengrass_core_sdk_java/sdk
$ ls
GreengrassJavaSDK-1.1.jar
```

SDK の jar ファイルを発見しました。このファイルをプロジェクトに持ってきます。

IntelliJ でプロジェクトを右クリックし、 [New] -> [Directory] を開き、 `libs` と入力して [OK] を選びます。

{{<figure src="/img/ss/idea7.png" class="center">}}

ファイルマネージャー等で先ほどの GreengrassJavaSDK-1.1.jar をこのディレクトリに移動します。

移動したら、以前と同じ手順で Gradle のリフレッシュを行います。するとコンパイルエラーが消えるはずです。

{{<figure src="/img/ss/idea8.png" class="center">}}

消えたあとは、この Gradle ツールウインドウで [Tasks] -> [other] とたどり、[buildZip] をダブルクリックします。
すると、 `build/distributions/<PROJECT>-1.0-SNAPSHOT.zip` というファイルが出来上がるはずです。

{{<figure src="/img/ss/idea9.png" class="center">}}

このファイルの位置を覚えておきます。

### Lambda 関数のデプロイ

再び Greengrass コンソールに戻ります。左側のメニューから [Greengrass] にある [グループ] を選びます。

{{<figure src="/img/ss/greengrass9.png" class="center" width="80%">}}

対象グループを選び、左側のメニューから [Lambda] を選び、右上の [Lambda の追加] を選びます (または [最初の Lambda を追加])。

{{<figure src="/img/ss/greengrass10.png" class="center" width="80%">}}

Greengrass グループへの Lambda の追加という画面になるので、 [新しい Lambda の作成] を選びます。

{{<figure src="/img/ss/greengrass11.png" class="center" width="80%">}}

AWS Lambda の画面が開きました。
適当な名前を入れ、ランタイムに [Java 8] を選択、ロールは [テンプレートから新しいロールを作成] を選び、ロール名も適当に入力します。
ポリシーテンプレートは空のままで構いません。
終わったら、 [関数の作成] を選びます。

{{<figure src="/img/ss/lambda2.png" class="center" width="80%">}}

関数コードをアップロードするときが来ました。
[ハンドラ] に **demo.HelloWorld::handleRequest** と入力し、 [アップロード] ボタンで先ほどの zip を選択します。

{{<figure src="/img/ss/lambda3.png" class="center" width="80%">}}

アップロードしたら、画面右上の [保存] を選びます。
続いて [アクション] から [新しいバージョンを発行] を選びます。

{{<figure src="/img/ss/lambda4.png" class="center" width="80%">}}

バージョンに名前を付けます。ここは適当に v1 などと入力して [発行] を選びます。

{{<figure src="/img/ss/lambda5.png" class="center" width="80%">}}

先ほどの Greengrass コンソールのほうに戻り、今度は [既存の Lambda の使用] を選びます。

{{<figure src="/img/ss/greengrass11.png" class="center" width="80%">}}

できたてほやほやの Lambda を選び、 [次へ] に進みます。

{{<figure src="/img/ss/greengrass12.png" class="center" width="80%">}}

できたてほやほやのバージョンを選び、 [完了] に進みます。

{{<figure src="/img/ss/greengrass13.png" class="center" width="80%">}}

Lambda のページに戻ってきました。

ここでいったん、 Lambda を編集します。対象の Lambda の行の [・・・] から [設定の編集] を選びます。

{{<figure src="/img/ss/greengrass14.png" class="center" width="80%">}}

メモリ制限を **64 MB** に、Lambda のライフサイクルを **存続期間が長く無制限に稼働する関数にする** にして [更新] を選びます。

{{<figure src="/img/ss/greengrass15.png" class="center" width="80%">}}

次に、画面左手の :arrow_left: より戻って [サブスクリプション] メニューを選び、 [サブスクリプションの追加] を選びます。

{{<figure src="/img/ss/greengrass18.png" class="center" width="80%">}}

**ソース**に先ほどデプロイした Lambda を、**ターゲット**に **IoT Cloud** を選びます。
Lambda のバージョンが複数ある場合は、最新のものを選択する必要があります (つまり、バージョンが上がったらこの設定もやり直しです！`$LATEST` バージョンエイリアスも使えません)。

{{<figure src="/img/ss/greengrass19.png" class="center" width="80%">}}

トピックのフィルターには、コードで指定したトピック **hello/world** を入力し、 [次へ] を選びます。

{{<figure src="/img/ss/greengrass20.png" class="center" width="80%">}}

問題なさそうなら内容を確認して、 [完了] を選びます。

{{<figure src="/img/ss/greengrass21.png" class="center" width="80%">}}

さあいよいよデプロイです。Raspberry Pi 3 は起きていますか？ `greengrassd` は生きていますか？指差喚呼したら、 [アクション] から [デプロイ] を選びます。

{{<figure src="/img/ss/greengrass16.png" class="center" width="80%">}}

[自動検出] を選びます。

{{<figure src="/img/ss/greengrass17.png" class="center" width="80%">}}

緑色のマークと共に「正常に完了しました」と表示されれば成功です。

それでは Raspberry Pi 3 から上がってくるメッセージを見てみましょう！
AWS IoT コンソールの左側のメニューから [テスト] を選び、MQTT クライアントを開きます。

[トピックのサブスクリプション] という入力欄があるので、コードと Greengrass グループに設定したトピック名 **hello/world** を入力します。
最後に [トピックに発行] を選びます。

{{<figure src="/img/ss/awsiot19.png" class="center" width="80%">}}

5秒ごとにメッセージが上がってくれば成功です！あがってこなかったら、なにかがおかしいです。後述のトラブルシューティングを参考にしてみてください。

{{<figure src="/img/ss/awsiot20.png" class="center" width="80%">}}

### 仕上げ

うまく動くことを確認したら、 GGC を systemd 化しましょう。早速設定ファイルを準備します <sup>[参考文献6]</sup>。

`/etc/systemd/system/greengrass.service`:

```
[Unit]
Description=greengrass daemon
After=network.target

[Service]
ExecStart=/greengrass/ggc/core/greengrassd start
Type=simple
RestartSec=2
Restart=always
User=root
PIDFile=/var/run/greengrassd.pid

[Install]
WantedBy=multi-user.target
```

サービスとして起動する前に、既に動いている `greengrassd` を手動で止めておきましょう。

```console
$ cd /greengrass/ggc/core
$ sudo ./greengrassd stop
Stopping greengrass daemon of PID: 599
Waiting..
Stopped greengrass daemon, exiting with success
```

正常に終了しましたね。それではサービスを有効化して起動しましょう。

```console
$ sudo systemctl enable greengrass
$ sudo systemctl start greengrass
```

状態は `service greengrass status` で確認できます。
これで、この Raspberry Pi 3 は再起動してもずっと GGC と一緒です。めでたしめでたし。

### トラブルシューティング

切り分けの基本は事実の把握から。

まず、デプロイがうまくいっているかどうかは `/greengrass/ggc/deployment/lambda` 以下にデプロイしたブツが一式揃っているかで確認することができます。
デプロイしたものが動いているかどうかの確認には、以下のログファイルをチェックしてみると良いでしょう。

- /greengrass/ggc/var/log/crash.log
- /greengrass/ggc/var/log/system/runtime.log
- /greengrass/ggc/var/log/system/java_runtime.log
- /greengrass/ggc/var/log/system/ ... (ほかにもたくさん)
- /greengrass/ggc/var/log/user/ ... (ローカル Lambda 関数のログ)

なお、私の環境では `crash.log` に次のログが 5 秒おきに書き込まれますが、 MQTT 自体は正しく動いていました。

```console
Java runtime initialization failed: com.amazonaws.greengrass.runtime.exception.InvalidHandlerException: Handler class was not found
```

紛らわしいので勘弁してほしいものです。

あと、 `gradlew buildZip` したときの Java のバージョンにも要注意です。最近は JDK9 や 10 や 11 を入れている人も多いかもしれません。ちゃんと Java 8 のクラスファイルを吐く必要があります。
万全を来すなら、以下のようにして `gradlew buildZip` をしましょう (例は macOS の場合)。

```console
$ JAVA_HOME=`/usr/libexec/java_home -v 1.8` ./gradlew clean buildZip
```

このほか、今回公式サポート外の環境 (Raspbian Stretch など) で行っています。
他にも環境依存の様々な問題が発生する可能性があるので、正しい環境も用意してそちらで動くか試す等の切り分けが必要になるかもしれません。
そして、こうした環境依存の切り分けのためにも、 SD カードは常に最低 2 枚は用意しておきたいところです。

Stay tuned & Happy hacking!

#### 参考文献

1. [よくある質問 &#x2013; AWS Greengrass | AWS](https://aws.amazon.com/jp/greengrass/faqs/)
2. [Kotlin/Native - Kotlin Programming Language](https://kotlinlang.org/docs/reference/native-overview.html)
3. [AWS Greengrass とは &#x2013; AWS Greengrass](https://docs.aws.amazon.com/ja_jp/greengrass/latest/developerguide/)
4. [AWS Greengrass ドキュメント](https://aws.amazon.com/jp/documentation/greengrass/)
5. [AWS Greengrass アプリケーションのトラブルシューティング  - AWS Greengrass](https://docs.aws.amazon.com/ja_jp/greengrass/latest/developerguide/gg-troubleshooting.html)
6. [greengrass systemd](https://gist.github.com/matthewberryman/fa21ca796c3a2e0dfe8224934b7b055c)
