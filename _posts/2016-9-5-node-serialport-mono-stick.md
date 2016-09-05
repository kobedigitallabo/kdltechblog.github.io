---
layout: post
title: node-serialportでMONO Stickの通信を取得してみる
published: true
---

村岡です。[KDL新事業創造係IoT班](http://www.kdl.co.jp/special/iot.html)ではいろいろなコネクテッドデバイスのプロトタイピングを行っています。
その中でケースに応じて様々なタイプの無線モジュールを試してみたりしてるんですが、最近はよく[TWE-Lite](http://mono-wireless.com/jp/products/TWE-LITE/index.html)を使っている気がします。
TWE-Liteは[ハルロック](http://morning.moae.jp/lineup/362)にもよく登場するのでその影響かもしれないんですが、簡単なことならプログラミングレスでいろいろできたり、技適通ってたりするのでささっとプロトタイピングしてみるのにはちょうどいいんですよね。
あとUSBスティックの[MONO Stick](http://mono-wireless.com/jp/products/MoNoStick/index.html)があるのもいい。PCにつないでデータモニタリングするのに便利です。

今回はこのMONO StickをPCに接続してMONO STickと[TWE-Lite DIP](http://mono-wireless.com/jp/products/TWE-Lite-DIP/index.html)を通信させ、node.jsでMONO Stickの通信を取得してみます。個人的にIoTはJSに集約されればいいと思ってるのでnode.js以外はあり得ません。  
node.jsではUSBシリアル通信のためのモジュール[node-serialport](https://github.com/EmergingTechnologyAdvisors/node-serialport)を使います。

# 準備

まずMONO StickをPCに接続します。MONO Stickのポート名は`/dev/tty.usbserial-XXXXXXX`だとします。  
TWE-Lite DIPの電源を入れておき、MONO Stickと通信している状態にしておきます。

npmでnode-serialportをインストールします。

```
mkdir twelite-monostick && cd $_
npm init
npm install serialport --save
```

# コード

```javascript
var serialPort = require('serialport');

var portName = '/dev/tty.usbserial-XXXXXXX';

var options = {
    baudRate: 115200,  // ボーレートは115200
    dataBits: 8,
    parity: 'none',    // パリティなし
    stopBits: 1,
    flowControl: false,  // フロー制御なしにしとく
    parser: serialPort.parsers.readline("\r\n")   // パースは\r\nで
};

var sp = new serialPort.SerialPort(portName, options);

sp.on('data', function(input) {
    try {
        console.log(input);
    } catch(e) {
        return;
    }
});
```

上記を`index.js`として保存します。

# 実行

```
node index.js
```

![スクリーンショット 2016-06-18 13.07.22.png (124.9 kB)](https://img.esa.io/uploads/production/attachments/3505/2016/06/18/10856/1b54a7a7-dd31-4677-8186-127b0d3070c5.png)

データの内容は[このページ](http://mono-wireless.com/jp/products/MoNoStick/monitor.html)のデータの読み方を参照してください。

TWE-Lite DIPのデフォルトのファームだと一定間隔で上記のような固定長のデータが降ってきます。node.jsのコードも簡単ですね。MONO StickをラズパイとかにつけてTWE-Liteのベースクライアントにするのも簡単にできそうです。いろいろと想像が広がりますね。
