---
title: "みちびき対応 GPS モジュール PA6C を Mbed で使う"
date: 2017-11-07T20:34:00+09:00
draft: false
categories: ["iot"]
tags: [
  "mbed",
  "node-red",
  "c++",
  ]
---

[前回の記事](/2017/11/06/using-esp-wroom-02-with-mbed/)に引き続き Mbed ネタです。今回ご紹介するのはコチラ！

{{< figure src="/img/ss/pa6c-breakout.jpg" title="PA6C Breakout基板 (実装済み)" class="center" width="40%" >}}

みちびき対応と謳われている GPS モジュール PA6C (FGPMMOPA6C) が載った基盤です。Running Electronics で \3,580 で売っているのを買ってきました ([www.runele.com/ca2/7/p-r-s/](http://www.runele.com/ca2/7/p-r-s/))。ブレッドボードに挿せる足も生えていて使いやすい基盤ですね。

前回同様、mbed LPC1768 に繋いでみます。

| mbed LPC1768 | FGPMMOPA6C     |
| ------------ | -------------- |
| VOUT (3.3V)  | VCC (#1)       |
| P28 (TX)     | RX             |
| P29 (RX)     | TX             |
| GND          | GND (#3 or #4) |

シリアル通信と電源供給だけ。まるで USB だ。

{{< figure src="/img/photo/20171106-2240.jpg" title="配線完了 (前回の写真を再掲)" class="center" width="80%" >}}

早速出力を見てみたいと思います。下のコードは後で送信処理がしやすいようレコード1本単位でループが回るようにしています。

```c
#include "mbed.h"

Serial gps(p28, p27);       // TX, RX
Serial pc(USBTX, USBRX);    // TX, RX
DigitalOut led1(LED1);

int main() {
    pc.baud(115200);
    pc.printf("PA6C DEMO\n");
    char gpsout[1024];
    while (1) {
        gpsout[0] = '\0';
        while (1) {
            char c = gps.getc();
            char s[2];
            s[0] = c;
            s[1] = '\0';
            strcat(gpsout, s);
            if (c == '\n') {
                break;
            }
        }
        pc.printf(gpsout);
        led1 = !led1;
    }
}
```

そうそう、Windows の方は mbed 用シリアルポートドライバ導入をお忘れなく。

> Windows serial configuration - Handbook | Mbed
> 
> https://os.mbed.com/handbook/Windows-serial-configuration

ではコンパイル、書き込み、ターミナル繋いでリセットボタン！

{{< figure src="/img/ss/coolterm2.png" title="PA6C からの出力" class="center" >}}

なにやら CSV っぽい見た目のフォーマットでデータが吐かれていきます。やたらゼロが多いような・・・？

このフォーマットは **NMEA 0183** というフォーマット<sup>[参考文献3]</sup>で、お目当ての座標データは `$GPGGA` から始まるレコードに含まれている模様。後に続くカンマ区切りの値がそれぞれ以下を意味します。

- UTC 時刻 (HHmmss.SS)
- 緯度 (Google Map とかで使う緯度の 100 倍の値)
- 緯度の符号 (N or S)
- 経度 (Google Map とかで使う緯度の 100 倍の値)
- 経度の符号 (E or W)
- (以下略)

値がなかったりゼロが並んでたりするのは、まだ計測できていないことを示しています。屋内ではなかなか時間がかかる模様...

このままデータを眺めていても退屈なので、このデータをビジュアライズしてくれるツールを使ってみましょう。以下のページから **Mini GPS tool (windows only)** をダウンロードします。

> Resources | Adafruit Ultimate GPS | Adafruit Learning System
> 
> https://learn.adafruit.com/adafruit-ultimate-gps/resources

起動するとこんな感じです。Comport は [Auto Detect] を押すと自動でセットされます。

{{< figure src="/img/ss/minigps1.png" title="Mini GPS Tool その1" class="center" >}}

やはり、まだ電波をキャッチできていないようです。でもしばらく待つと、12 というマークがついて日付・時刻が入りました。

{{< figure src="/img/ss/minigps2.png" title="Mini GPS Tool その2" class="center" >}}

参考文献4によると、12番くんはアメリカの GPS、2R-16M のようです。みちびき1号機は193番、2号機は194番なので、まだ PA6C の底力は発揮できていませんね。GPS は最低でも衛星3機＋時間差を補正するためのもう1機の**合計4機**がないと測位できないため、この状態では測れません。外に出たくないならば、辛抱強く待ちましょう。

・・・と思ったのですが、窓をあけてちょこっと突き出しただけでこの通り！

{{< figure src="/img/ss/minigps3.png" title="Mini GPS Tool その3" class="center" >}}

193番、みちびき1号機登場！座標もドンピシャで出ました (モザイクいれていますが)。

ここまで確認できれば、あとは煮るなり焼くなりです。とりあえず前回の記事の Wi-Fi モジュールを使って [VULTR VPS 東京リージョン](https://www.vultr.com/?ref=7053029) に置いた Node-RED に送りつけてみました。NMEA 0183 フォーマットのまま送りつけているので、パースもクラウド上です。こんな感じで lat, lon を取り出すことができました。

{{< figure src="/img/ss/nodered1.png" title="Node-RED" class="center" >}}

理想は mbed 側で NMEA パースし、さらに LoRaWAN 送信を見越すと11バイト以内 (例えば Float x2 など) に収めたいところですが、それは今後の課題ということで、今日はここまで。

Stay tuned & Happy hacking!

#### 参考文献

1. [FGPMMOPA6C Breakout 基板 ユーザーズマニュアル](http://runele.jp/pa6cbo/pa6cbreak.pdf)
2. [GPSモジュールを使ってシリアル通信をやってみる。 | Mbed](https://os.mbed.com/users/yamato/notebook/gps_Serial/)
3. [NMEA 0183 Format](http://www2.nc-toyama.ac.jp/~mkawai/lecture/radionav/nmea0183.html)
4. [各国の測位衛星｜技術情報｜みちびき（準天頂衛星システム：QZSS）公式サイト - 内閣府](http://qzss.go.jp/technical/satellites/index.html)
