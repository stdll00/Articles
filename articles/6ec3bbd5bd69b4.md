---
title: "RM Mini3を使ってFireTVのリモコンから中華プロジェクターの電源を入れるようにした話"
emoji: "📽" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["iot","python","raspberrypi"] # タグ。["markdown", "rust", "aws"]のように指定する
published: true # 公開設定（falseにすると下書き）
---
## 概要
FireTVリモコンを使い**FireTV非対応の**プロジェクターの電源の操作をできるようにしました。
あまり安定してないので2，3回ボタンを押さないと動かないことがあります。ちなみにRM Mini3は受信中でも普通に動くようなのでRM Mini3はこれ専用にする必要はなさそうです。

対応端末であれば
![](https://storage.googleapis.com/zenn-user-upload/vfjalllqt3tqmctsmqrdi9mw5eqv)
となるところ、RM Mini3を挟んで
![](https://storage.googleapis.com/zenn-user-upload/l7a57pk6gj1c00v1p7easu11z5bb)
という流れで信号を変換して送ります。

頑張れば適当なリモコンの信号で全く関係ないものを動かしたり、プロジェクターをつけるときに部屋の照明を落としたりいろいろできそうです。

### 必要なもの
- RM Mini3(1,2,4でも多分動きます) (Aliexpressで2000円くらいで売ってます) eRemote miniとして日本版が売られています
- FireTV Stick or その他IRが出せるなにか
- テレビ、プロジェクターなど FireTVで対応済みのものは普通に設定すれば使えます
- RaspberryPiなどのPythonが動かせるLan内のサーバー

### 設定方法
- RM Mini3のセットアップ
- FireTVで 設定 ->機器のコントロール->機器の管理->IR(赤外線)オプション->IRプロフィール->IRプロフィールを変更->IRプロフィールを検索-> **00334 を検索、設定**
- `pip3 install broadlink`と`pip3 install git+https://github.com/elupus/irgen.git@master`
- IR信号のバイナリを記録してソースコードのPROJECTOR_COMMANDを書き換える(後述)
- 以下のソースコードがずっと走るようにする(私の場合、RaspberryPi4で `sudo pm2 start ir_remote_convert.py --name ir_conv  --interpreter python3`で実行しました)

##### IR信号のバイナリ取得方法
とりあえずソースコードを実行すると`signal`(整数のタプル)、`ir_packet`(バイト列)が出ます。
`ir_packet`はそのまま送信にも使えます。 
```
        print(signal)
        print(ir_packet)
```
### ソースコード


仕組みは
FIRE TV リモコンの信号を受信 → IRをバイナリで受け取る   
受け取ったIRのバイナリをNECフォーマットとして解析して信号を取り出す。この信号のフォーマットは
`(16, -8, 1, -3, 1)` なら 16単位時間オン→8単位時間オフ(オフの部分にはマイナスが付きます)→1単位時間オン→3単位時間オフ→1単位時間オン といった感じです。本当は誤り訂正などの要素もありますが無視して誤り訂正も含め一致したもののみを扱います。本当は誤り訂正などもありますがとりあえず無視して完全一致のみを判定しています 

→ 受け取ったコマンドに応じてプロジェクターの信号を送る

私のプロジェクターは音量が100段階あるので1回ボタンを押したら8回信号を送るようにしています

```
#! /usr/bin/python3

import irgen
import broadlink
import time
from threading import Thread
import base64
import subprocess

_ = irgen  # pip3 install git+https://github.com/elupus/irgen.git@master

class IRSignal:
    def __init__(self, signal):
        self.signal = signal

    @classmethod
    def init_from_raw(cls, raw_format_array):
        t = (raw_format_array[0] - raw_format_array[1]) / 24
        # t = 562
        _signal = tuple(round(v / t) for v in raw_format_array[:-1])
        return cls(_signal)

    @classmethod
    def from_raw_string(cls, s: str):
        raw_format_array = [int(float(v.replace("+", ""))) for v in s.strip().split(" ")]
        return cls.init_from_raw(raw_format_array)

    @classmethod
    def from_packet(cls, ir_packet: bytes):
        s = subprocess.getoutput(
            "irgen -i broadlink_base64 -d {} -o raw".format(base64.b64encode(ir_packet).decode()))
        return cls.from_raw_string(s)


PROJECTOR_COMMAND = {
    'POWER': b'&\x00P\x00\x00\x01(\x92\x14\x11\x14\x11\x13\x11\x15\x11\x14\x10\x14\x11\x14\x11\x14\x11\x146\x145\x146\x146\x136\x15\x10\x146\x146\x14\x11\x14\x10\x155\x146\x145\x15\x10\x14\x11\x15\x10\x146\x146\x13\x11\x14\x11\x15\x10\x155\x137\x145\x14\x00\x05D\x00\x01(I\x14\x00\r\x05\x00\x00\x00\x00\x00\x00\x00\x00',
    'VOL_UP': b"&\x00P\x00\x00\x01(\x92\x15\x10\x14\x11\x14\x11\x14\x11\x15\x10\x15\x10\x14\x11\x14\x11\x146\x137\x136\x146\x137\x14\x11\x136\x146\x137\x136\x155\x14\x11\x13\x12\x14\x11\x13\x12\x13\x12\x14\x11\x13\x12\x13\x12\x145\x146\x137\x136\x146\x13\x00\x05F\x00\x01'I\x15\x00\r\x05\x00\x00\x00\x00\x00\x00\x00\x00",
    'VOL_DOWN': b"&\x00P\x00\x00\x01'\x93\x14\x11\x13\x12\x13\x13\x12\x12\x13\x12\x13\x12\x13\x12\x13\x12\x137\x136\x146\x137\x136\x14\x11\x146\x137\x136\x146\x137\x13\x12\x13\x12\x14\x11\x136\x15\x10\x14\x11\x14\x11\x14\x11\x146\x137\x136\x14\x11\x146\x14\x00\x05E\x00\x01)H\x14\x00\r\x05\x00\x00\x00\x00\x00\x00\x00\x00",
    'SOURCE': b'&\x00P\x00\x00\x01(\x92\x15\x10\x14\x11\x14\x11\x14\x11\x14\x11\x14\x11\x14\x11\x14\x11\x137\x136\x155\x146\x146\x13\x12\x145\x146\x137\x145\x14\x11\x146\x146\x14\x11\x136\x14\x11\x14\x11\x14\x11\x146\x14\x11\x13\x12\x137\x14\x11\x136\x14\x00\x05E\x00\x01(I\x14\x00\r\x05\x00\x00\x00\x00\x00\x00\x00\x00',
    "ENTER": b"&\x00P\x00\x00\x01(\x93\x13\x12\x13\x12\x13\x12\x13\x12\x13\x12\x14\x10\x14\x11\x15\x10\x146\x146\x136\x146\x146\x13\x12\x137\x136\x14\x11\x146\x137\x13\x12\x13\x12\x14\x11\x14\x11\x14\x11\x136\x14\x11\x14\x11\x146\x137\x145\x146\x137\x14\x00\x05E\x00\x01'I\x14\x00\r\x05\x00\x00\x00\x00\x00\x00\x00\x00",
    "MUTE": b'&\x00T\x00\x14\x11\x14\x11\x14\x11\x14\x11\x13\x12\x13\x12\x14\x11\x13\x12\x137\x145\x155\x137\x145\x14\x11\x155\x146\x14\x11\x13\x12\x14\x11\x136\x14\x11\x15\x10\x14\x11\x14\x11\x146\x137\x136\x14\x11\x146\x146\x145\x146\x14\x00\x05D\x00\x01)H\x14\x00\x0cR\x00\x01(H\x15\x00\r\x05\x00\x00\x00\x00',
}

# IR CODE 00334
FIRE_TV_SIGNAL_TO_COMMAND = {
    (16, -8, 1, -3, 1, -1, 1, -1, 1, -3, 1, -1, 1, -1, 1, -1, 1, -3, 1, -1, 1, -3, 1, -3, 1, -1, 1, -3, 1, -3, 1, -3, 1,
     -1, 1, -3, 1, -1, 1, -1, 1, -1, 1, -1, 1, -3, 1, -3, 1, -1, 1, -1, 1, -3, 1, -3, 1, -3, 1, -3, 1, -1, 1, -1, 1, -3,
     1, -71, 16, -4, 1):
        [PROJECTOR_COMMAND['POWER']],
    (16, -8, 1, -3, 1, -1, 1, -1, 1, -3, 1, -1, 1, -1, 1, -1, 1, -3, 1, -1, 1, -3, 1, -3, 1, -1, 1, -3, 1, -3, 1, -3, 1,
     -1, 1, -3, 1, -3, 1, -3, 1, -1, 1, -3, 1, -3, 1, -1, 1, -1, 1, -1, 1, -1, 1, -1, 1, -3, 1, -1, 1, -1, 1, -3, 1, -3,
     1, -71, 16, -4, 1):
        [PROJECTOR_COMMAND['VOL_UP']] * 8,
    (16, -8, 1, -3, 1, -1, 1, -1, 1, -3, 1, -1, 1, -1, 1, -1, 1, -3, 1, -1, 1, -3, 1, -3, 1, -1, 1, -3, 1, -3, 1, -3, 1,
     -1, 1, -1, 1, -1, 1, -1, 1, -3, 1, -1, 1, -3, 1, -1, 1, -1, 1, -3, 1, -3, 1, -3, 1, -1, 1, -3, 1, -1, 1, -3, 1, -3,
     1, -71, 16, -4, 1):
        [PROJECTOR_COMMAND['VOL_DOWN']] * 8,
    (16, -8, 1, -3, 1, -1, 1, -1, 1, -3, 1, -1, 1, -1, 1, -1, 1, -3, 1, -1, 1, -3, 1, -3, 1, -1, 1, -3, 1, -3, 1, -3, 1,
     -1, 1, -3, 1, -3, 1, -1, 1, -1, 1, -1, 1, -3, 1, -3, 1, -1, 1, -1, 1, -1, 1, -3, 1, -3, 1, -3, 1, -1, 1, -1, 1, -3,
     1, -71, 16, -4, 1):
        [PROJECTOR_COMMAND['MUTE']]
}

devices = broadlink.discover(timeout=5)
if len(devices) == 0:
    exit(1)
device = devices[0]
device.auth()
print("auth complete")

while True:
    device.enter_learning()
    try:
        time.sleep(0.5)
        ir_packet = device.check_data()
    except broadlink.exceptions.ReadError:
        continue


    def execute(ir_packet):
        signal = IRSignal.from_packet(ir_packet).signal
        print(signal)
        print(ir_packet)
        if signal in FIRE_TV_SIGNAL_TO_COMMAND:
            print("Send signal")
            for ir_packet in FIRE_TV_SIGNAL_TO_COMMAND[signal]:
                device.send_data(ir_packet)
                time.sleep(0.1)


    Thread(target=execute, args=(ir_packet,)).start()

```

### 参考にしたサイト
NEC Format : https://garretlab.web.fc2.com/arduino/make/infrared_sensor_nec_format/  
Broadlink RM mini3(黒豆)をRaspberry Piで動かす https://www.taneyats.com/entry/rm_mini3_on_raspberrypi
