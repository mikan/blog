---
title: "10 ドル Linux ボード La Frite に Debian 10 をインストールして Phoronix Test Suite で性能を計測する"
date: 2019-11-23T22:15:00+09:00
categories: ["iot"]
tags: [
  "raspberrypi",
  "benchmark",
]
---

La Frite は 2018/10/12 に Kickstarter に登場した 10 ドルで Raspberry Pi 3 を上回る性能を叩き出す (と謳われる) Linux ボードのファウンディングプロジェクトです。

{{<figure src="/img/photo/20191123-1653.jpg" class="center" width="80%">}}

La Frite は実際には 10 ドルの RAM 512MB 版と 15 ドルの RAM 1GB 版があり、 15 ドル版は Android も動くそうです。
開発元の Libre Computer Project は既に一回り大きい Le Potato というボードを販売しており、La Frite はその小型版という位置づけとなっています。

ファウンディングが成功し Kickstarter キャンペーンは終わりましたが、現在は一般販売が始まり、現在は前者が **20 ドル**、後者が **25 ドル**で販売中です (このせいでなんだかブログ記事がタイトル詐欺っぽくなっちゃいました:innocent:)。

> Libre Computer Board AML-S805X-AC &ndash; Love Our Pi - The Destination for Single Board Computers
> 
> https://www.loverpi.com/collections/la-frite/products/libre-computer-board-aml-s805x-ac

今回私は 15 ドル版をプレッジし、物が手元に届いたのは 2019/7/24。コスト削減のため製造ベンダーを変えたりして遅くなったそうです。届いたあとは Debian 9 をインストールしたり、 eMMC カードを追加購入してそちらから起動してみたりなどしてしばらく遊んだあと、最近 Debian 10 のイメージも公開されました。

### Raspberry Pi との違い

まずは La Frite のスペックを Raspberry Pi 3 と比較してみましょう。

| Item      | La Frite (512MB/1GB)            | Raspberry Pi 3 Model B          |
| --------- | ------------------------------- | ------------------------------- |
| CPU Model | Amlogic S805X (**28nm** Proc.)  | Broadcom BCM2837 (40nm Proc.)   |
| CPU Spec  | Cortex-A53 Quad-Core 1.2GHz     | Cortex-A53 Quad-Core 1.2GHz     |
| GPU       | Mali-450 Penti-Core             | Broadcom VideoCore IV           |
| RAM       | 512MB/1GB **DDR4**              | 1GB LPDDR2                      |
| Storage   | External USB, External **eMMC** | **MicroSD** Slot, External USB  |
| Power-in  | MicroUSB                        | MicroUSB                        |
| USB 2.0   | 2 Ports                         | **4 Ports**                     |
| HDMI      | **HDMI 2.0** with 1080P Output  | HDMI 1.4 with 1080P Output      |
| LAN       | 100BASE-TX (**Direct**)         | 10BASE-T, 100BASE-TX (USB)      |
| Wi-Fi     | No                              | **Yes** (802.11 b/g/n 2.4 GHz)  |
| Bluetooth | No                              | **Yes** (Bluetooth 4.1, BLE)    |
| GPIO      | 40-pin                          | 40-pin                          |
| Sensors   | **IR Sensor**                   | -                               |
| Size      | 65mm x 56mm (*)                 | 85.6 mm x 56.5 mm               |
| Unit Cost | $10/$15 -> **$20/$25**          | $35                             |

(*) La Frite のサイズの情報がなかったので実測しました。Raspberry Pi より一回り小さくてかわいい感じです。NanoPi と比べてしまうとまだ大きいですが・・・。

値段差の最大要因は無線 LAN の有無と思われます (Kickstarter の FAQ より)。
La Frite は既に FCC と CE の認証は得ていますが、無線 LAN を加えると各国のハードウェア認証を得る手続きが必要になり、世界中で売るために必要なコストが上がるそうです。

使う上で大きく異なるのは起動方法です。
Raspberry Pi は MicroSD スロット刺したカードから OS を起動するのが最も一般的な方法ですが (USB からのブートも可能)、La Frite は MicroSD スロットはありません。
まずは USB メモリ等の Mass-Storage Class の USB 機器にイメージを焼き、そこから起動します。

また、裏面に eMMC コネクタがあり、そこから OS を起動することもできます。
その場合もいったんは USB で起動し、ツールを使って eMMC にデータをコピーすることで eMMC からの起動がはじめてできるようになるという仕組みです。

本番稼働するシステムを構築する際は、USB の遅さ、ポートが 1 つふさがってしまう不便さ、USB コネクタの基板実装の強度などを考えても eMMC が良さそうです。
公式サイトで購入した eMMC 基板は、本体とカラーリングもマッチしていい感じです。

{{<figure src="/img/photo/20191123-1654.jpg" class="center" width="80%">}}

### セットアップ手順

まずはイメージをダウンロードします。以下の URL に最新の Debian イメージがあります。
GUI が不要なら Headless を、GUI がほしければ XFCE 版か LXDE 版を選びます。

> http://share.loverpi.com/board/libre-computer-project/libre-computer-board/image/debian/README.php

私がダウンロードしたのは Headless 版 (`libre-computer-aml-s805x-ac-debian-buster-headless-mali-4.19.64+-2019-08-05.zip`) で、ファイルサイズは 670.1MB です。
混雑していてダウンロードが遅いときは、気長にコーヒーでも飲んで待ちましょう。

ダウンロードが完了したら展開 (`unzip`) します。
私がダウンロードしたファイルの場合、展開すると 1.88GB になりました。
USB へ焼くツールは Win32DiskImager か balenaEtcher が推奨されています。
私は macOS に対応している balenaEtcher を用いました。

> balenaEtcher - Home
>
> https://www.balena.io/etcher/

こんな感じです。 Flash!

{{<figure src="/img/ss/etcher1.png" class="center" width="80%">}}

USB ブートの準備ができたら、USB ポートに接続します。
ちなみにドキュメントには一番 IR sensor (赤外線センサー) から遠い USB ポートに刺すと書いていてあります (理由は不明)。
ほかに、以下を接続します:

- USB キーボード
- HDMI モニタ
- LAN ケーブル
- MicroSD 電源

電源を刺した瞬間に起動が開始します。ここは Raspberry Pi と同じですね。

ブートシーケンスが完了しログインプロンプトが表示されたら、以下を入力してログインします:

- Username: `libre`
- Password: `computer`

ログイン出来ましたか？おめでとうございます :tada:

デフォルトで SSH サーバーが有効になっており、 (`hostname -I` 等で) IP がわかればリモートログインできます。

`uname` コマンドを見ると Kernel は 4.19.64 が、アーキテクチャは `aarch64` になっていることがわかります。
ようするに 64bit ARM (ARMv8) に最適化された OS ということです！
Raspberry Pi の Raspbian は CPU が 64bit 対応でも GPU ドライバの都合やプログラムの互換性の都合などに縛られて 32bit のままなので、
La Frite はソフト環境面でもアドバンテージがあるといえるでしょう。

```console
$ uname -a
Linux libre-computer 4.19.64+ #60 SMP PREEMPT Mon Aug 5 14:22:48 EDT 2019 aarch64 GNU/Linux
```

### eMMC を使う

La Frite 用の eMMC は次の場所で購入することができます。8, 16, 32, 64, 128GB がラインナップされています。

> Libre Computer Board eMMC 5.x Module &ndash; Love Our Pi - The Destination for Single Board Computers
>
> https://www.loverpi.com/products/libre-computer-board-emmc-5-x-module

USB ケーブルが付属していますが、私は一度も使っていません。
eMMC をベンダー提供のツールでプログラミングする際に使うようです。

手に入れたら裏面にはめ込み、いったん今まで通り USB 起動します。

起動し、ログインしたら、 `lc_distro_transfer` というコマンドを使って書き込み内容を確認していきます。

```console
$ sudo lc_distro_transfer
Available Vendor/Model List:
libre-computer/aml-s805x-ac
```

表示されたモデルを引数に追加します。

```console
$ sudo lc_distro_transfer libre-computer/aml-s805x-ac
Available Block Device List:
/dev/mmcblk0 emmc
/dev/sda
```

eMMC のデバイスパスが `/dev/mmcblk0` であることがわかります。
パスを引数に追加します。

```console
$ sudo lc_distro_transfer libre-computer/aml-s805x-ac /dev/mmcblk0
Transferable Distro List:
lc-debian-10-headless
```

ディストリビューションの一覧が表示されます。
これですべての情報が揃いました。
ディストリビューション名を引数に追加します。

```console
$ lc_distro_transfer libre-computer/aml-s805x-ac /dev/mmcblk0 lc-debian-10-headless
```

デバイスの準備 (パーティション作成やフォーマット等) が自動で実行され、最後にデータ転送が行われ、最後に `DEVICE /dev/mmcblk0 READY!` と表示されて終了します。

おわったら、シャットダウンします。

```
$ sudo shutdown -h now
```

電源と USB を抜いて、再び電源を入れると・・・ eMMC から OS が起動するようになります。

### ベンチマーク

Libre Computer が公開している Geekbench の結果が次の URL にあります。

> brcm Raspberry Pi 3 vs Libre Computer AML-S805X-AC - Geekbench Browser
>
> https://browser.geekbench.com/v4/cpu/compare/8763939?baseline=10596298

しかし表示内容が実物と違っていたり (CPU 周波数とか)、Android のバージョンが異なっているなどの差異があるので、あまり鵜呑みにできません。

ならば自分で試してみようということで、手元にある Raspberry Pi 3 Type B と比較してみることにしました。
とはいえ、ベンチマークを本当に正確に行うのは難しいのと、個体差などもありえます。私の情報も鵜呑みにしないでください。

まずは `uname -a` を確認しておきます。

La Frite:

```console
Linux libre-computer 4.19.64+ #60 SMP PREEMPT Mon Aug 5 14:22:48 EDT 2019 aarch64 GNU/Linux
```

Raspberry Pi 3:

```console
Linux ykm4-raspbian 4.19.75-v7+ #1270 SMP Tue Sep 24 18:45:11 BST 2019 armv7l GNU/Linux
```

ベンチマークツールは Phoronix Test を選定してみました。

```console
$ wget http://phoronix-test-suite.com/releases/repo/pts.debian/files/phoronix-test-suite_9.0.1_all.deb
$ sudo apt install ./phoronix-test-suite_9.0.1_all.deb
$ phoronix-test-suite
// 利用許諾、匿名レポート送信の可否を回答
```

たくさんのベンチマークテストやスイートが登録されていますが、人気度やシンプルさを重視して Gzip 圧縮テストをチョイスしてみました<sup>[参考文献1]</sup>。
実行してみます。

```
$ phoronix-test-suite benchmark pts/compress-gzip
```

ずいぶんと時間がかかりそうだったので、この間に魚焼きグリルで焼きなすを2本を作りました。

作り終えるとちょうど終わったところだったので、 OpenBenchmark.org にアップロードするよう指示。果たして結果は・・・？

じゃかじゃん！

La Frite:

{{<figure src="/img/ss/pts1.png" class="center" width="80%" link="https://openbenchmarking.org/result/1911238-AS-LAFRITECO12">}}

Raspberry Pi 3:

{{<figure src="/img/ss/pts2.png" class="center" width="80%" link="https://openbenchmarking.org/result/1911230-AS-RPI3BCOMP23">}}

**僅差で La Frite の勝ち！**

開発元の Geekbench 疑ってすみません。たしかに La Frite は速いようです。
ほかにもいろいろテストすると異なる傾向が出るかもしれませんが、プロセッサ以外ではメモリは DDR2 (Raspberry Pi 3) と DDR4 (La Frite) と二世代の差を付けていますし、ディスク性能は MicroSD 対 eMMC の対決となってしまい、結果は目に見えています。

というわけで、ものすごく安価なのに、性能も上々な Raspberry Pi 3 オルタナティブであることを確認することができました。
あとはハウジング (ケース、シャーシ) のラインナップが充実すると嬉しいのですが、残念ながらこれに限っては Raspberry Pi が圧倒的に強いですね。
公式からはアルミニウムフレームケースが $5 でラインアップされています。eMMC を買うときににでも一緒に買い物かごにいれてはいかが。

> La Frite &ndash; Love Our Pi - The Destination for Single Board Computers
>
> https://www.loverpi.com/collections/la-frite

Happy hacking!

#### 参考文献

1. [OpenBenchmarking.org - pts / Phoronix Test Suite Test Profiles](https://openbenchmarking.org/tests/pts&s=p)
