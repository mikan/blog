---
title: "Raspberry Pi Pico と SCD41 で作る CO2 表示器"
date: 2021-12-26T05:20:00+09:00
draft: false
categories: ["iot"]
tags: [
  "python",
  "raspberrypi",
  "sensing"
  ]
---

{{<figure src="/img/photo/20211220-0105.jpg" class="center" width="50%">}}

年末が近づいてきてだいぶ寒い日が続くようになってきましたが、寒くなってくると換気がおっくうになってお部屋の CO2 濃度が上がりがちです。
CO2 濃度が上がると人体には集中力低下などの影響があると言われており <sup>[参考文献1]</sup>、脳みそを使って知的でクリエイティブな仕事をする我々プログラマーにとって見えない大敵です。
さらに、コロナ禍においては屋内で人が密集してしまう環境では十分な換気が感染リスク減少に寄与することが言われており <sup>[参考文献2]</sup>、換気の指標として CO2 濃度の計測は有用です。

ところがその CO2 濃度を計測するセンサーには粗悪品が多いという情報があります。電通大の調査によると、粗悪な CO2 センサーは消毒用アルコールに強く反応してしまうそうです <sup>[参考文献3]</sup>。
しっかりモニタリングしたければ高価な CO2 計測器を買えばよいのですが、できれば安価に実現したいもの。ということで電子工作界隈で流行り出したのが CO2 センサーの自作。
人気なのは安価なこちらのセンサー、Zhengzhou 社製 MH-Z19 シリーズ。写真の MH-Z19C は秋葉原の秋月で ¥2,500 ぐらい、激安といっていいでしょう <sup>[参考文献4]</sup>。

{{<figure src="/img/photo/20211226_0331.jpg" class="center" width="50%">}}

NDIR (非分散型赤外) 方式というのを採用していて、測定範囲は 400〜5000ppm、誤差 ± (50ppm + 5% reading value) とスペックも上々。金ピカでゴツい見た目もなかなか。ちなみに端っこに張り付いている紙切れはフィルターなので、剥がしたりしてはいけません。

ただし今回紹介するのはこれではなく **Sensirion SCD41** です。
Sensirion といえば SHT シリーズなど CMOS を使った独自技術による高精度な温湿度センサーが有名ですが、こちらはその CO2 バージョンのようなものです。値段は MH-Z19 シリーズほどではないですが CO2 センサーとしてはやはり結構安い部類、そしてセンシリオン公式 <sup>[参考文献5]</sup> のほか最近 Seeed <sup>[参考文献6]</sup> や Adafruit <sup>[参考文献7]</sup> からも評価ボードが発売され入手もしやすいです。スペック上も、測定範囲こそ MH-Z19C と同じですが、誤差は ± (40ppm + 5% reading value) とより高精度になっています <sup>[参考文献8]</sup>。しかもこのセンサー、 CO2 だけでなく温度・湿度も1モジュールでできてしまいます。冬場部屋で加湿器を運用している私にとって高精度の湿度のモニタリングも同時に達成したい。というわけで今回は SCD41 を使った表示器を製作することに決定、まずはブレッドボードに配線してみました。

{{<figure src="/img/photo/20211219_2223.jpg" class="center" width="50%">}}

マイコンは今ホットな Raspberry Pi Pico を採用。LCD は I2C で手軽に使えて ¥550 と安価な AQM1602 のキット <sup>[参考文献9]</sup> を利用しています。
ちなみにこのキット、なかなか細かいハンダ付けが必要なので、しっかり換気して集中力を高めた状態で作業しましょう (笑)

配線はこんな感じです。上の写真とは AQM1602 が左右逆になっているのでご注意ください。

{{<figure src="/img/diagram/rpi-pico-scd41-wiring.png" class="center" width="70%">}}

プログラムはせっかくの Pi Pico なので MicroPython をチョイスしました。
SCD41 のドライバーは Adafruit が公開している CircuitPython 向けのものを MicroPython 化して対応しました。

> adafruit/Adafruit_CircuitPython_SCD4X: CircuitPython/Python driver for Sensirion SCD40 & SCD41
>
> https://github.com/adafruit/Adafruit_CircuitPython_SCD4X

ただしキャリブレーション等の機能はまだ移植していません。

AQM1602 のドライバーは秋月電子通商の商品サイトにある Arduino 向けサンプルプログラムを MicroPython 化して対応しました。

> Ｉ２Ｃ接続小型キャラクタＬＣＤモジュール　１６×２行　３．３Ｖ／５Ｖ: ディスプレイ･表示器 秋月電子通商-電子部品・ネット通販
>
> https://akizukidenshi.com/catalog/g/gP-08779/

完成したプログラム (`main.py`) は以下のようになりました。ちょっと長いですが全文を貼り付けます。

```python
import machine
import time
from micropython import const


class SCD4X:
    """
    Based on https://github.com/adafruit/Adafruit_CircuitPython_SCD4X
    Copyright (c) 2021 ladyada for Adafruit Industries
    MIT License
    """

    DEFAULT_ADDRESS = 0x62
    DATA_READY = const(0xE4B8)
    STOP_PERIODIC_MEASUREMENT = const(0x3F86)
    START_PERIODIC_MEASUREMENT = const(0x21B1)
    READ_MEASUREMENT = const(0xEC05)

    def __init__(self, i2c_bus, address=DEFAULT_ADDRESS):
        self.i2c = i2c_bus
        self.address = address
        self._buffer = bytearray(18)
        self._cmd = bytearray(2)
        self._crc_buffer = bytearray(2)

        # cached readings
        self._temperature = None
        self._relative_humidity = None
        self._co2 = None

        self.stop_periodic_measurement()

    @property
    def co2(self):
        """Returns the CO2 concentration in PPM (parts per million)
        .. note::
            Between measurements, the most recent reading will be cached and returned.
        """
        if self.data_ready:
            self._read_data()
        return self._co2

    @property
    def temperature(self):
        """Returns the current temperature in degrees Celsius
        .. note::
            Between measurements, the most recent reading will be cached and returned.
        """
        if self.data_ready:
            self._read_data()
        return self._temperature

    @property
    def relative_humidity(self):
        """Returns the current relative humidity in %rH.
        .. note::
            Between measurements, the most recent reading will be cached and returned.
        """
        if self.data_ready:
            self._read_data()
        return self._relative_humidity

    def _read_data(self):
        """Reads the temp/hum/co2 from the sensor and caches it"""
        self._send_command(self.READ_MEASUREMENT, cmd_delay=0.001)
        self._read_reply(self._buffer, 9)
        self._co2 = (self._buffer[0] << 8) | self._buffer[1]
        temp = (self._buffer[3] << 8) | self._buffer[4]
        self._temperature = -45 + 175 * (temp / 2 ** 16)
        humi = (self._buffer[6] << 8) | self._buffer[7]
        self._relative_humidity = 100 * (humi / 2 ** 16)

    @property
    def data_ready(self):
        """Check the sensor to see if new data is available"""
        self._send_command(self.DATA_READY, cmd_delay=0.001)
        self._read_reply(self._buffer, 3)
        return not ((self._buffer[0] & 0x03 == 0) and (self._buffer[1] == 0))

    def stop_periodic_measurement(self):
        """Stop measurement mode"""
        self._send_command(self.STOP_PERIODIC_MEASUREMENT, cmd_delay=0.5)

    def start_periodic_measurement(self):
        """Put sensor into working mode, about 5s per measurement"""
        self._send_command(self.START_PERIODIC_MEASUREMENT, cmd_delay=0.01)

    def _send_command(self, cmd, cmd_delay=0.0):
        self._cmd[0] = (cmd >> 8) & 0xFF
        self._cmd[1] = cmd & 0xFF
        self.i2c.writeto(self.address, self._cmd)
        time.sleep(cmd_delay)

    def _read_reply(self, buff, num):
        self.i2c.readfrom_into(self.address, buff, num)
        self._check_buffer_crc(self._buffer[0:num])

    def _check_buffer_crc(self, buf):
        for i in range(0, len(buf), 3):
            self._crc_buffer[0] = buf[i]
            self._crc_buffer[1] = buf[i + 1]
            if self._crc8(self._crc_buffer) != buf[i + 2]:
                raise RuntimeError("CRC check failed while reading data")
        return True

    @staticmethod
    def _crc8(buffer):
        crc = 0xFF
        for byte in buffer:
            crc ^= byte
            for _ in range(8):
                if crc & 0x80:
                    crc = (crc << 1) ^ 0x31
                else:
                    crc = crc << 1
        return crc & 0xFF  # return the bottom 8 bits


class AQM1602:
    """
    Based on https://akizukidenshi.com/catalog/g/gP-08779/
    """

    DEFAULT_ADDRESS = 0x3E

    def __init__(self, i2c_bus, address=DEFAULT_ADDRESS):
        self.i2c = i2c_bus
        self.address = address
        time.sleep_ms(100)
        self.write_cmd(0x38)
        time.sleep_ms(20)
        self.write_cmd(0x39)
        time.sleep_ms(20)
        self.write_cmd(0x14)
        time.sleep_ms(20)
        self.write_cmd(0x73)
        time.sleep_ms(20)
        self.write_cmd(0x56)
        time.sleep_ms(20)
        self.write_cmd(0x6C)
        time.sleep_ms(20)
        self.write_cmd(0x38)
        time.sleep_ms(20)
        self.write_cmd(0x01)
        time.sleep_ms(20)
        self.write_cmd(0x0C)
        time.sleep_ms(20)

    def write_data(self, data):
        self.i2c.writeto_mem(self.address, 0x40, bytes([data & 0xFF]), addrsize=8)
        time.sleep_ms(1)

    def write_cmd(self, cmd):
        self.i2c.writeto_mem(self.address, 0x00, bytes([cmd & 0xFF]), addrsize=8)
        time.sleep_ms(1)

    def print(self, line_no, lin):
        buf = bytearray(lin)
        if len(buf) <= 0:
            return
        if len(buf) > 16:
            buf = buf[0:16]
        if line_no == 0:
            self.write_cmd(0x01)
            self.write_cmd(0x80)
        else:
            self.write_cmd(0x02)
            self.write_cmd(0xC0)
        for idx in range(0, len(buf)):
            self.write_data(buf[idx])


if __name__ == "__main__":
    led = machine.Pin(25, machine.Pin.OUT)
    uart = machine.UART(1, 9600, tx=machine.Pin(4), rx=machine.Pin(5))

    # I2C0: SCD4X
    i2c0 = machine.I2C(0, sda=machine.Pin(0), scl=machine.Pin(1), freq=100000)
    print("i2c0 scan result:", i2c0.scan())
    scd4x = SCD4X(i2c0)
    scd4x.start_periodic_measurement()

    # I2C1: LCD
    i2c1 = machine.I2C(1, sda=machine.Pin(2), scl=machine.Pin(3), freq=100000)
    print("i2c1 scan result:", i2c1.scan())
    lcd = AQM1602(i2c1)

    seq = 0
    print("seq,co2,temperature,humidity")
    lcd.print(0, "SCD4X CO2 Sensor")
    lcd.print(1, "Initializing...")

    while True:
        time.sleep(5)
        seq = seq + 1
        co2_ppm = scd4x.co2
        temp_deg = scd4x.temperature
        humidity_percent = scd4x.relative_humidity
        print(seq, co2_ppm, temp_deg, humidity_percent, sep=",")
        if co2_ppm >= 2000:
            led.on()
        else:
            led.off()
        lcd.print(0, "%dppm" % co2_ppm)
        lcd.print(1, "%.1f%cC %.1f%%" % (temp_deg, 13, humidity_percent))
```

main 処理にあるように、2,000ppm を超えると Pico の基板にある LED が点灯するように仕込んであります。また計測データは UART 経由で CSV 形式で出力する機能もつけてあります。

Pico へのプログラムの書き込みは Thonny を使いました。Pico を繋いだ状態で起動すると、MicroPython 用ファームウェアの書き込こんどく？みたいなダイアログが出てきて、ワンクリックでセットアップできる、とても便利なツールです。

> Thonny, Python IDE for beginners
>
> https://thonny.org/

試行錯誤するときは Thonny のエディタに打ち込んだプログラムを Run ボタンを押すだけでそのまま Pico 上で動きます。
Pico 本体にプログラムを配置して走らせたい場合は、保存先として Pico を選び `main.py` という名前で保存します。

一度 Pico 上に配置してしまえば、その後は Pico 上にある `main.py` を Thonny で開いていじることもできます。なおその際には一度 Stop ボタンを押して今走っているプログラムを止めておく (シェルに `>>>` が出ている状態にする) 必要があります。

{{<figure src="/img/ss/thonny1.png" class="center" width="50%">}}

{{<figure src="/img/ss/thonny2.png" class="center" width="50%">}}

息を吹きかけたりして CO2・温度・湿度が変化することを確認してみましょう。

一通り動作を確認したら、今度はブレッドボードではなくもうちょっと使いやすいようにユニバーサル基板にしたくなりました。
そして完成したのが冒頭の写真です (再掲)。

{{<figure src="/img/photo/20211220-0105.jpg" class="center" width="50%">}}

用意した部材はこちら:

- ユニバーサル基板 2.54mm ピッチ 両面スルーホール Bタイプ
  - https://akizukidenshi.com/catalog/g/gP-03232/
- アクリル版 B基板用
  - https://akizukidenshi.com/catalog/g/gP-10243/
- M3 ネジ・ナット (x4)
  - https://akizukidenshi.com/catalog/g/gP-02743/
- M3 スペーサー 20mm (x4)
  - https://akizukidenshi.com/catalog/g/gP-07570/
- 分割ロングピンソケット・・・SCD41 接続用 (ボードによって4〜8ピン使用) ※基板に直付けする場合は省略可能
  - https://akizukidenshi.com/catalog/g/gC-05779/
- ピンソケット 2x20 (x2)・・・Pico 接続用 (2列目は不使用) ※基板に直付けする場合は省略可能
  - https://akizukidenshi.com/catalog/g/gC-00085/
- 丸ピン IC ソケット・・・AQM1602 接続用 (4ピンのみ使用) ※基板に直付けする場合は省略可能
  - https://akizukidenshi.com/catalog/g/gP-01591/
- タクトスイッチ・・・リセットボタン用
  - https://akizukidenshi.com/catalog/g/gP-03647/
- すずめっき線・はんだ

写真の作例は 2,000ppm 超えインジケーターをより目立たせるため Pico 上の LED ではなく基板に置いた赤色 LED に切り替えていますが、それ以外は上に書いた配線図通りです。

表のようす:

{{<figure src="/img/photo/20211220_0051.jpg" class="center" width="50%">}}

裏のようす:

{{<figure src="/img/photo/20211220_0105.jpg" class="center" width="50%">}}

なかなかパンクな感じに仕上がりました。
7 セグ LED を並べたりすればもっと存在感ある感じになったのですが、電気を消した部屋で煌々と光るのもなんかあれなのでテキスト LCD に落ち着いています。

なお本記事に掲載したプログラムや配線図は GitHub でも公開しています。
不具合や改善点を見つけた際などは是非 issue や PR をお送りください。お待ちしております。

> https://github.com/mikan/rpi-pico-scd4x

Happy hacking!

#### 参考文献

1. [教室環境の質が児童の体調と集中力に与える影響に関する実態調査](https://www.jstage.jst.go.jp/article/aije/77/676/77_533/_article/-char/ja/)
2. [CiNii 論文 - コロナウィルスの感染対策に有用な室内環境に関連する研究事例の紹介](https://ci.nii.ac.jp/naid/130007884051/)
3. [【ニュースリリース】安価で粗悪なCO<sub>2</sub>センサの見分け方 ～５千円以下の機種、大半が消毒用アルコールに強く反応～│電気通信大学](https://www.uec.ac.jp/news/announcement/2021/20210810_3625.html)
4. [ＣＯ２センサーモジュール　ＭＨ－Ｚ１９Ｃ: センサ一般 秋月電子通商-電子部品・ネット通販](https://akizukidenshi.com/catalog/g/gM-16142/)
5. [Evaluation Kit SEK-SCD41 | Sensirion](https://www.sensirion.com/en/environmental-sensors/evaluation-kit-sek-environmental-sensing/evaluation-kit-sek-scd41/)
6. [Grove - CO2 & Temperature & Humidity Sensor - SCD41](https://www.seeedstudio.com/Grove-CO2-Temperature-Humidity-Sensor-SCD41-p-5025.html)
7. [Adafruit SCD-41 - True CO2 Temperature and Humidity Sensor [STEMMA QT / Qwiic] : ID 5190 : $59.50 : Adafruit Industries, Unique & fun DIY electronics and kits](https://www.adafruit.com/product/5190)
8. [CO2センサー 「SCD4x」 | Sensirion](https://www.sensirion.com/jp/environmental-sensors/carbon-dioxide-sensors/carbon-dioxide-sensor-scd4x/)
9. [Ｉ２Ｃ接続小型キャラクタＬＣＤモジュール（１６×２行・３．３Ｖ／５Ｖ）ピッチ変換キット: ディスプレイ･表示器 秋月電子通商-電子部品・ネット通販](https://akizukidenshi.com/catalog/g/gK-08896/)
