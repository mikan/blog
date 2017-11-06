---
title: "ESP-WROOM-02 を Mbed で使う"
date: 2017-11-06T22:49:31+09:00
draft: false
categories: ["iot"]
tags: [
  "hackathon",
  "mbed",
  "network",
  "c++",
  ]
---

先日 (2017/10/21)、長野県の伊那市で「[第3回 伊那市 LoRaWAN ハッカソン](https://uhuru.connpass.com/event/65550/)」というのが開かれまして、初参加してきました。

6人1チームで全6チームで伊那市の課題を解決するアイデアソン＋ LoRaWAN でデモを開発するというハッカソンで、私は「チーム1」に配属、凄腕メンバーに囲まれたおかげで2位を受賞することができました。チーム1で提案したシステムは、伊那市の豊かな山地の道なき道をゆく登山者、林業従事者、きのこハンター、けものハンター、徘徊老人たちにデバイスを持たせ、踏破実績を収集・蓄積して山林ルートマップを作成、地域の様々な課題解決やサービスに役立てられるプラットフォームを作る！というものでした。そして今回作ったデモシステムは mbed に GPS とボタンを実装、LoRaWAN で割と高頻度で送信し、最終的に超イケメン BI ツール [MotionBoard](http://www.wingarc.com/product/motionboard/) でビジュアライズするというものです (ドローンは構想のみ)。

{{< figure src="/img/ss/inahack-team1-5.png" title="InaHack チーム1 システム構成" class="center" width="80%" >}}

開発の部で私が主に担当したのはデータの中継 (と加工) を行う [Enebular](https://enebular.com/) (Node-RED プロバイダー) のフローで、一通り組んだあとはこんな感じになっていました。

{{< figure src="/img/ss/inahack-enebular.png" title="InaHack チーム1 Enebular フロー" class="center" width="80%" >}}

ハードのほうも少しお手伝いして、こんな感じの見た目になりました。徘徊老人がこれ持って歩いてたらめっちゃロックだよね！:metal:

{{< figure src="/img/photo/20171022-1555.jpg" title="デモ筐体" class="center" width="80%" >}}

#### 本題

さて、ここからが今回の記事の本題なのですが (**前置きが長い**)、ハッカソンが終わったあと、このコンセプトを自分の手元でも再現したいと思いまして、まずは LoRaWAN に相当する部分を WLAN に置き換えて試そうと、こんなものを買ってきました。

{{< figure src="/img/ss/esp-breakout.jpg" title="ESP-WROOM-02ピッチ変換済みモジュール" class="center" width="40%" >}}

Switch Science でたったの \909 で売っています！([ssci.to/2347](http://ssci.to/2347)) 技適認証済です！ただし足はついていないので、一緒に買ってきてハンダ付けする必要があります。まあ、これぐらいはこの記事に興味があるお兄ちゃん達は余裕だよね。

ハンダ付けが終わったら、配線！なのですが、mbed のサイトで公開されている [esp8266-driver](https://os.mbed.com/teams/ESP8266/) などのプログラムは `AT` コマンドというのをシリアルで投げることでチップに指示を送る仕組みを使っており、出荷時のファームウェアではこれに対応していないとのこと。なのでまずはファームウェアアップグレードを行います。なお、今回は純正の SDK を用いて書き換えを行うため、ファームウェアを書き換えても技適の認証には影響を与えないそうです<sup>[参考文献3]</sup>。安心してください。

今回用意したのは mbed LPC1768 で、PC からシリアル入出力を行うため、シリアルポートドライバを導入しておく必要があります (Windows の場合)。

> Windows serial configuration - Handbook | Mbed
> 
> https://os.mbed.com/handbook/Windows-serial-configuration

それでは、参考文献1を読みながらファーム書き込み用の配線です (太字はあとで配線変えます)。

| mbed LPC1768    | ESP-WROOM-02    |
| --------------- | --------------- |
| VOUT (3.3V)     | 3V3             |
| VOUT (3.3V)     | EN (CH_EN)      |
| VOUT (3.3V)     | IO2 (GPIO2)     |
| **GND**         | **IO0 (GPIO0)** |
| P9 (TX)         | RXD (RX)        |
| P10 (RX)        | TXD (TX)        |
| **VOUT (3.3V)** | **RST**         |
| GND             | GND             |

{{< figure src="/img/photo/20171106-2240.jpg" title="配線完了 (余計なの写ってるけどそれはあとで別記事にする)" class="center" width="80%" >}}

そういえばこのチップ、そうとうな大飯ぐらいみたいで、mbed が再起動を繰り返す場合は、電流不足です (私の mbed コレクションの中でもいける個体といけない個体あり ← 改造のせい)。そういうときはこんな感じで 3.3V を他の mbed から救援してみましょう (別に mbed じゃなくても良いのだけど、自己責任で)。

{{< figure src="/img/photo/20171106-2236.jpg" title="3.3V 給電を別の mbed から救援中の図" class="center" width="80%" >}}

準備ができたらファームウェアをダウンロードします。

> ESP8266EX Resources | Espressif Systems
> 
> http://espressif.com/en/products/hardware/esp8266ex/resources

ダウンロードするのは以下の２つ:

- SDKs & Demos 欄にある「**ESP8266 NONOS SDK**」(私が利用したのは  V2.1.0)
- Tools 欄にある「**Flash Download Tools (ESP8266 & ESP32)**」(私が利用したのは V3.6.1.0)

mbed に電源を入れ、Flash Download Tool (ESPFlashDownloadTool_v3.6.1.0.exe) を起動し、ESP8266 DownloadTool ボタンを押すと、ファームウェアとアドレスを選ぶ画面になるので、説明書 (ESP8266_NONOS_SDK/bin/at/README.md) を読みながら指示どおり入力していきます。

なお、今回使ったチップは Switch Science のサイトに「フラッシュ 2MB」(= 16Mb) と書いてあったので、以下の値 (抜粋) を入力します (フラッシュサイズが異なるチップも多数流通しているようなので、よくご確認を！)。

```
### Flash size 16Mbit-C1: 1024KB+1024KB
    boot_v1.2+.bin              0x00000
    user1.2048.new.5.bin        0x01000
    esp_init_data_default.bin   0x1fc000 (optional)
    blank.bin                   0xfe000 & 0x1fe000
```

以下が書き込み中の画面 (下戴中)。START 押したとき FLASH SIZE の選択値が 16Mbit-C1 から 16Mbit に勝手に変わったけどバグかな？ :innocent:

{{< figure src="/img/ss/esp-download.png" title="ESP8266 DOWNLOAD TOOL V3.6.1.0" class="center" width="40%" >}}

書き込みが終わったら動作チェックです。素晴らしいツールが転がっていたので紹介します。

> ESP8266-configuration-mbed-LPC1768 - a mercurial repository | Mbed
> 
> https://os.mbed.com/users/4180_1/code/ESP8266-configuration-mbed-LPC1768/

すごい！追加ライブラリなしで生で AT コマンドゴリゴリ叩いてコンソールにデバッグログ出してくれる！こりゃ使うしかありません。

おっとその前に再配線です。太字の部分をつけかえます (今更ですが、P9-P11 を使っているのは [LPC11U24](https://os.mbed.com/platforms/mbed-LPC11U24/) でも検証したかっただけで、他の TX/RX でも OK です)。

| mbed LPC1768    | ESP-WROOM-02    |
| --------------- | --------------- |
| VOUT (3.3V)     | 3V3             |
| VOUT (3.3V)     | EN (CH_EN)      |
| VOUT (3.3V)     | IO2 (GPIO2)     |
| **VOUT (3.3V)** | **IO0 (GPIO0)** |
| P9 (TX)         | RXD (RX)        |
| P10 (RX)        | TXD (TX)        |
| **P11**         | **RST**         |
| GND             | GND             |

GPIO0 が High か High かで通常モードかファーム書き込みモードか切り替わるようです。

ツールのソースコードも合わせて変更します。

```diff
- Serial esp(p28, p27); // tx, rx
+ Serial esp(p9, p10); // tx, rx
- DigitalOut reset(p26);
+ DigitalOut reset(p11);
```

では適当なターミナルエミュレータ (Tera Term とか) で結果を見てみましょう！Baudrate はソースコードに合わせて 115200 に設定します。

{{< figure src="/img/ss/coolterm1.png" title="CoolTerm" class="center" width="80%" >}}

(TONKOTSU と SHOYU が我が家の AP です)

ばっちりですね！もしデバッグに AT コマンドが表示されなかったり、あるいは OK とか返って来ていない場合は、何かがおかしいです。すぐに電源を外して色々確認してみましょう。

ここまでつながればあとは好きなプロトコルで通信するだけですね！ESP8266 対応の HTTP GET する素晴らしいサンプルがあるのでご紹介しておきます。

> SITB_HttpGetSample - a mercurial repository | Mbed
> 
> https://os.mbed.com/users/jksoft/code/SITB_HttpGetSample/

次回は (mbed の横に鎮座していたのに触れなかった) みちびき対応の GPS チップ PA6C をハックします。

Stay tuned & Happy hacking!

#### 参考文献

1. [Firmware Update -  | Mbed](https://os.mbed.com/teams/ESP8266/wiki/Firmware-Update)
2. [ESP-WROOM-02ピッチ変換済みモジュール《フル版》 - スイッチサイエンス](http://ssci.to/2347)
3. [ESP-WROOM-02のファームウェアを書き換えた場合、技適はどうなるのか | スイッチサイエンス マガジン](http://mag.switch-science.com/2016/01/20/esp-wroom-02_telec/)
