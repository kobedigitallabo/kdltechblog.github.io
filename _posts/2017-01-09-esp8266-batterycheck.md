---
published: true
layout: post
title: ESP8266とAmbientで電池の放電電圧を可視化してみる
tags:
  - iot
  - arduino
  - esp8266
  - ambient
  - mini4w
---
# はじめに
こんにちは。ナカニシです。
[KDL新事業創造係IoT班](http://www.kdl.co.jp/special/iot.html)ではいろいろなコネクテッドデバイスのプロトタイピングを行っています。
今回はニッケル水素充電池用に放電機を作成したのでインターネットにつなげてみました。

## 動機
簡易な装置でバッテリーの放電状況を可視化することで、これまでは数万円するような放電機でしか調べる事ができなかった、放電特性を調べる事が可能になるのでは無いかと思いIoT放電機を作ってみました。
ESP8266を利用してインターネットに接続し、[Ambient – IoTクラウドサービス](https://ambidata.io/)にて可視化してみました。
ESP8266はアナログポートが一つしか無いので、Arduinoのアナログポートを利用して、Arduino側で処理した電圧データをESP8266経由でクラウドにアップして可視化することに決めたので、ESP8266 と Arduino 間をI2C通信で接続に挑戦してみました。
今流行の、Arduino と ESP8266 を胸元で「ふんっ！」てして、バッテリーの放電状況を可視化してみました。

## 対象読者
* Arduino ESP8266 に興味がある人
* ESP8266 でアナログデータを取得したい人
* Mini-Zやミニ四駆のパワーソースに興味がある人
* 電池を育てて見たい人
* マッチドバッテリーを作りたい人
* タミヤ信者

## 開発環境
* ProductName: Mac OS X El Capitan
* ProductVersion: 10.11.6
* Arduino IDE 1.6.13
* ESP8266 version:1.1.0.0(May 11 2016 18:09:56) SDK version:1.5.4(baaeaebb)
* Arduino Pro Mini 3.3V

# 本ミッションのシナリオ
バッテリーの種類は様々で乾電池からリチウム・ポリマーなど多くの種類が存在しています。
また取り出せる電圧も様々で1.5Vや24Vなど種類が様々となります。
今回は手始めに、比較的扱いやすい、単三電型のニッケル・水素充電池(NiMH)を使ってみることにします。

ざっくりとした手順としては･･･
1. ニッケル水素充電池の放電機の作成
1. Arduino側での電圧測定をする
1. ESP8266側で取得したデータをクラウドにアップする
1. クラウド上でグラフを設定して確認する
となります。

## 放電機を作成する
単に電池を計測しても、その変化が現れにくく計測する意味も残量表示ぐらいで意味が無いので、状況変化が現れやすく意味があるデータを取得するために、放電機を作成し、放電状況のデータを取得して可視化することにしました。

雑に説明すると充電池は正しい放電することでリフレッシュされ、充電池の劣化を防ぐ事が可能となります。
しかし放電しすぎると逆に性能劣化をおこす事につながり、また充電しすぎるのも問題で性能を劣化させたり、破壊につながり爆発や発火したりするために、丁度良い充電、放電を行う必要があります。

放電機は[自己電源方式、オートカット電圧可変、ニッケル水素充電池・単セル放電器の製作 \[自作放電器\]](http://www.kansai-event.com/kinomayoi/disc/discH.html)を参考に作成しました。
今回作成した放電機は部品点数が少なく低コストで作成が出来、かつ単セルで放電できて設定した電圧以下はオートカットされ過放電されないために、ほったらかしでも電池にダメージを与えない仕組みとなっています。

ついでに放電機には秋月電子で販売されていた、超小型２線式ＬＥＤデジタル電圧計を[秋月「超小型２線式ＬＥＤデジタル電圧計」を3線式（0V～）にする方法: エアーバリアブル ブログ](http://airvariable.asablo.jp/blog/2015/06/21/7677435)を参考に改造してスライドスイッチでそれぞれの電池容量を量る事ができるようにしておきました。

## ブレッドボードにてArduinoとESP8266をつなげてみる
次の図はブレッドボード上に仮組みしてしてみた際の回路です。電源は3.3vの入力、下部のバッテリーは計測対象の電池を想定しています。
アナログポート毎に１つの電池を計測しますので、実際の実装では放電機の電池ボックスの端子に接続することになります。

![バッテリーチェッカー_ブレッドボード.png](https://img.esa.io/uploads/production/attachments/3505/2017/01/09/10863/3a58a435-0f76-404e-b87a-f8e63d0319d1.png)

## Arduino側で電圧を測定し、I2CでESP8266側へ送信する
Arduinoのアナログポートを２つ利用して、2個の電池を同時に監視するようにしました。
取得したデータはI2C通信にてESP8266側に送信を行うことにします。
なおArduinoではsprintfでのfloatは使えないらしいです。その代わり、dtostrf()という関数があるのでそちらを利用しました。

ソースはgithubにアップしておきました。
[Ambient\_ESP8266\_Arduino\_BatteryCheck-slave](https://github.com/haru-kdl/Ambient_ESP8266_Arduino_BatteryCheck/tree/master/Ambient_ESP8266_BatteryCheck-slave)

## ESP8266側で取得したデータをクラウドにアップする
Arduino側より取得したデータをESP8266側で受信しインターネット経由で、Ambientに送信します。
Ambient側の設定は先に済ませておく必要があります。設定方法はAmbientのチュートリアル[Ambientを使ってみる – Ambient](https://ambidata.io/docs/gettingstarted/)を参考に設定しました。

ESP8266はI2CではMasterにしか設定出来ないために、ESP8266をMaster、ArduinoをSlaveとして作成しました。
ソースはgithubにアップしておきました。
[Ambient\_ESP8266\_Arduino\_BatteryCheck-master](https://github.com/haru-kdl/Ambient_ESP8266_Arduino_BatteryCheck/tree/master/Ambient_ESP8266_BatteryCheck-master)


# 作成した放電機とつなげてみる
Arduino Pro MiniのA0を放電機のバッテリーホルダー（１）の＋側へ、A1を放電機のバッテリーホルダー（２）の＋側へ接続し、放電機のGNDはArduinoのGNDと接続します。
Arduino Pro Miniのピンアサイン図を参考にESP8266とArduinoのSDA,SCL同士を接続します。
https://cdn.sparkfun.com/datasheets/Dev/Arduino/Boards/ProMini8MHzv1.pdf

放電機とユニバーサル基板上に実装したセンサ部をつなげてみたら接続したらあれげな感じに･･･
![IMG_5629.JPG (1.8 MB)](https://img.esa.io/uploads/production/attachments/3505/2017/01/07/10863/a4756363-4c46-4e2f-b326-498ffb00b9e7.JPG)

## ケースにいれたらもっとヤバイ感じになったｗ
青と赤の線どちらかを切る感じの雰囲気がでています。
![IMG_5652.JPG (1.7 MB)](https://img.esa.io/uploads/production/attachments/3505/2017/01/07/10863/eea2cdfd-7fd9-42c7-960d-ae807abec2a9.JPG)

## 起動させてみる
無事起動し、自動的にインターネットに接続されデータを送信し始めました。
![IMG_5651.JPG (1.5 MB)](https://img.esa.io/uploads/production/attachments/3505/2017/01/07/10863/9accfd4c-739b-488d-9e40-4f6516a53a52.JPG)

# ESP8266側で受信を確認してみる
無事にデータを受信できている事が確認できました。
<img width="942" alt="Ambient_ESP8266_BatteryCheck.png (217.6 kB)" src="https://img.esa.io/uploads/production/attachments/3505/2017/01/03/10863/1c82e0db-2c01-43f0-b062-ddc114e3ec76.png">

# Ambient上でグラフを設定して確認してみる
手元の乾電池とニッケル水素充電池をセットして確認すると電圧が確認できました。
放電を初めていないのに、何故か電池２の方の電圧が安定しない事が判明しました。これは電池の種類によって異なるかもしれません。
![image.png (81.0 kB)](https://img.esa.io/uploads/production/attachments/3505/2017/01/03/10863/7047722e-3db0-4e8e-b379-5650d8841989.png)

## 放電状況を確認する
手元のeneloopを満充電にし放電実験したところ両方電池共に0.98Vまで下がったところでオートカットが機能している事がわかり、放電機が正しく動作している事がわかりました。

![手元のeneloopを比較.jpg (305.5 kB)](https://img.esa.io/uploads/production/attachments/3505/2017/01/07/10863/b2e21705-78e5-4779-98c4-214d144c95c3.jpg)


# まとめ
放電機の作成までは順調でしたが、ESP8266を利用してArduinoとI2C通信を行うのが苦難の連続でした。
できる限りコストを削減し、モジュールも小型化したかったのでArduinoを選択したがメモリ不足だったり、I2C通信がうまくできなかったりと紆余曲折があり、非常に時間を掛けてしまったもののなんとかやりたい事が達成できて良かったです。
今回はESP8266側とArduinoの電圧をそろえたかったので、Arduino Pro Mini 3.3V版を採用しています。
ESP8266単体ではポートが不足する場合にはADCを利用してもいいのですが、サブマイコンにArduinoを利用してみるという安易な発想が実現できました。
中華製のArduino nanoなどは格安で販売されているので、多くのポートを利用する場合などはこの様な組み合わせもいいかもしれませんん。

# さいごに
最後にですが、秋月の店舗で販売されていた格安ケースのスペースに全てを納めることと、あえてマイコン部分を取り外し可能にしているのでケースの幅や高さがギリギリで納める事が難しかったです。

![IMG_5654.JPG (1.3 MB)](https://img.esa.io/uploads/production/attachments/3505/2017/01/07/10863/cc75ca2a-1634-4b21-b259-ff4fea3ffa8e.JPG)

放電機単体は外部電源が無くてもスタンドアロンで動作しますので、サーキットから自宅に戻る間に放電を行う事も可能となっています。
それでは良いIoTミニ四駆ライフをお過ごしください。


# 参考・関連
http://qiita.com/u_one/items/44b95274a40f9c40f4d0
http://tabrain.blog.so-net.ne.jp/2015-09-10
http://mixture-art.net/arduino%E3%81%A7i2c%E9%80%9A%E4%BF%A1%E3%82%92%E3%82%84%E3%82%8B%E9%9A%9B%E3%81%AE%E3%83%A1%E3%83%A2/
http://www.denshi.club/make/arduinoorg/wire.html
http://blog-yama.a-quest.com/?eid=970163
http://shirotsuku.sakura.ne.jp/blog/?p=710
https://tech-blog.cerevo.com/archives/859/
https://cdn.sparkfun.com/datasheets/Dev/Arduino/Boards/ProMini8MHzv1.pdf

