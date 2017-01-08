---
published: false
layout: null
title: ESP8266とambientで電池の放電電圧を可視化してみる
tags:
  - arduino
  - esp8266
  - maked
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
* 電源は3.3vの入力、下部のバッテリーは計測対象の電池を想定
* アナログポート毎に１つの電池を計測



"https://img.esa.io/uploads/production/attachments/3505/2017/01/09/10863/3a58a435-0f76-404e-b87a-f8e63d0319d1.png"

## Arduino側で電圧を測定し、I2CでESP8266側へ送信する
 にて電池電圧の計測が出来たので、このソースを利用してアナログポート２つを利用して、2個の電池を同時に監視するようにした。
取得したデータはI2C通信にてESP8266側に送信を行う。
なおArduino ではsprintfでfloatは使えないらしい。その代わり、dtostrf()という関数があるのでそちらを利用する。

```c++
// ESP8266とArduino側で通信を行う
// Arduino側 Slave側
#include <Wire.h>
#define I2CAddressESPWifi 6

const int BTTIN1 = A0;    // 電池をA0ピンに入力
const int BTTIN2 = A1;    // 電池をA1ピンに入力
const float VREF = 3.3;  // Analog reference voltage is 5.0V
const float R1 = 30.0;   // 30kOhm is used
const float R2 = 130.0;  // 130kOhm is used
char response[10] ="0.12|0.12";

void setup()
{
  Serial.begin(115200);
  pinMode(5,INPUT_PULLUP); //内部プルアップ抵抗を使う
  Wire.begin(I2CAddressESPWifi);
  Wire.onReceive(espWifiReceiveEvent);
  Wire.onRequest(espWifiRequestEvent);
}
void loop()
{
  delay(1);
}
void espWifiReceiveEvent(int count)
{
  //Serial.print("Received[");
  while (Wire.available())
  {
    char c = Wire.read();
    //Serial.print(c);
  }
  //Serial.println("]");
  //calc response.
  String ts = getData();
  strncpy(response,ts.c_str(),sizeof(response)-1);
}
void espWifiRequestEvent()
{
  /*Serial.print("Sending:[");
  Serial.print(response);
  Serial.println("]");*/
  Wire.write(response);
}
String getData(){
  char data1buf[12], data2buf[12];
  String str;
  double a1 = analogRead(BTTIN1); // アナログポートより電圧値を取得
  if(a1<0.1) a1 = 0.1; // 0除算を回避
  a1 *= VREF ;
  a1 /= 1024.0;
//    a1 /= R1;  // 分圧回路の場合の抵抗値
//    a1 *= R2; // 分圧回路の場合の抵抗値
  double a2 = analogRead(BTTIN2); // アナログポートより電圧値を取得
  if(a2<0.1) a2 = 0.1; // 0除算を回避
  a2 *= VREF ;
  a2 /= 1024.0;
  // 送信データ作成
  dtostrf(a1, 3, 2, data1buf);
  dtostrf(a2, 3, 2, data2buf);
  str = data1buf;
  str = str + "|";
  str = str + data2buf;
  //Serial.println(str);
  return(str);
}
```

# ESP8266側で取得したデータをクラウドにアップする
Arduino側よりデータを取得し、ambientに送信する。
ambient側の設定は先に済ませておく。設定方法は[#206:  HowTo/arduino/ESP8266をArduino化して温湿度と気圧を取得する](/posts/206) に記述を参考にした。
ESP8266とArduinoをI2C通信するためには、いろいろと手順が必要なので[#217:  HowTo/arduino/ESP8266でI2Cライブラリに振り回された](/posts/217) を元に作成した。
ESP8266はI2CではMasterにしか設定出来ないために、ESP8266をMaster、ArduinoをSlaveとして作成する。

```c++
#include <ESP8266WiFi.h>
#include <Wire.h>
#include "Ambient.h"

extern "C" {
#include "user_interface.h"
}

#define _DEBUG 1
#if _DEBUG
#define DBG(...) { Serial.print(__VA_ARGS__); }
#define DBGLED(...) { digitalWrite(__VA_ARGS__); }
#else
#define DBG(...)
#define DBGLED(...)
#endif /* _DBG */

#define I2CAddressESPWifi 6
#define LED 16
#define SDA 4
#define SCL 5
#define RDY 12
#define PERIOD 5

const char* ssid = "SSID-NO";
const char* password = "PASSWORD";
unsigned int channelId = 750;
const char* writeKey = "WRITEKEY";
WiFiClient client;
Ambient ambient;
int x=32;

void setup()
{
    wifi_set_sleep_type(LIGHT_SLEEP_T);

#ifdef _DEBUG
    Serial.begin(115200);
    delay(20);
#endif
    DBG("\r\nStart\r\n");

    //Wire.begin(SDA, SCL);
    Wire.begin();  //マスターに設定

    pinMode(LED, OUTPUT);

    WiFi.begin(ssid, password);
    DBG("wifi connect ");

    int i = 0;
    while (WiFi.status() != WL_CONNECTED) {
        delay(500);

        DBGLED(LED, i++ % 2);
        DBG(".");
    }

    DBGLED(LED, LOW);
    DBG("WiFi connected\r\n");
    DBG("IP address: ");
    DBG(WiFi.localIP());
    DBG("\r\n");

    //ambient接続開始
    ambient.begin(channelId, writeKey, &client);
}

void loop()
{
    myReadLine(1);
    delay(PERIOD * 1000);
}

void myReadLine(int a){
    char charA_out[80];
    int cnt_buf = 0;
    // Slave側へデータ作成実行
    Wire.beginTransmission(I2CAddressESPWifi);
    Wire.write(x);
    Wire.endTransmission();
    x++;
    if(x >= 2147483647) x = 32;
    delay(1);//Wait for Slave to calculate response.
    // Slave側データ読み込み
    Wire.requestFrom(I2CAddressESPWifi,9);
    DBG("Request Return:[");
    if(Wire.available() > 0){
      cnt_buf = Wire.available();  //受信文字数
      for (int iii = 0; iii < cnt_buf; iii++){
        delay(1);
        charA_out[iii] = Wire.read();
      }
      charA_out[cnt_buf] = '\0';  //終端文字
      String c = charA_out;
      DBG(c);
    }
    DBG("]"); DBG("\r\n");
    snd_ambient(charA_out);
}
void snd_ambient(char indata[80]){
    // ambientへの送信準備
    char *charB;
    charB = strtok(indata,"|");    //1つ目のトークン表示
    ambient.set(1, charB);   // ambientへデータ1送信
    DBG("data1:"); DBG(charB); DBG("\r\n");
    while(charB!=NULL){          //トークンがNULLになるまでループ
      charB = strtok(NULL,"|");  //2回目以降は第一引数はNULL
      if(charB!=NULL){
        ambient.set(2, charB);   // ambientへデータ2送信
        DBG("data2:"); DBG(charB); DBG("\r\n");
      }
    }
    DBG("\r\n");
    // ambientへデータ送信
    DBGLED(LED, HIGH);
    ambient.send();
    DBGLED(LED, LOW);
}
```
# 回路側の接続
* Arduino Pro MiniのA0を放電機のバッテリーホルダー（１）の＋側へ、A1を放電機のバッテリーホルダー（２）の＋側へ接続し、放電機のGNDはArduinoのGNDと接続します。
* Arduino Pro Miniのピンアサイン図を参考にESP8266とArduinoのSDA,SCL同士を接続します。

https://cdn.sparkfun.com/datasheets/Dev/Arduino/Boards/ProMini8MHzv1.pdf
![image.png (1.6 MB)](https://img.esa.io/uploads/production/attachments/3505/2017/01/07/10863/8dae0feb-75cb-4a02-8ca9-4fc6d645b005.png)




## 作成した放電機とつなげてみた
放電機とユニバーサル基板上に実装したセンサ部をつなげてみたら接続したらあれげな感じに･･･
![IMG_5629.JPG (1.8 MB)](https://img.esa.io/uploads/production/attachments/3505/2017/01/07/10863/a4756363-4c46-4e2f-b326-498ffb00b9e7.JPG)

## ケースにいれたらもっとヤバイ感じになったｗ
青と赤の線どちらかを切る感じですね。
![IMG_5652.JPG (1.7 MB)](https://img.esa.io/uploads/production/attachments/3505/2017/01/07/10863/eea2cdfd-7fd9-42c7-960d-ae807abec2a9.JPG)

## 全て接続してみる
無事起動して、自動的にインターネットに接続しデータを送信し始めた。
![IMG_5651.JPG (1.5 MB)](https://img.esa.io/uploads/production/attachments/3505/2017/01/07/10863/9accfd4c-739b-488d-9e40-4f6516a53a52.JPG)



# ESP8266側で受信を確認する
無事にデータを受信できている事が確認できた。
<img width="942" alt="Ambient_ESP8266_BatteryCheck.png (217.6 kB)" src="https://img.esa.io/uploads/production/attachments/3505/2017/01/03/10863/1c82e0db-2c01-43f0-b062-ddc114e3ec76.png">

# クラウド上でグラフを設定して確認する
電圧が確認できた。放電を初めていないのに、何故か電池２の方の電圧が安定しない事が分かった。これは電池の種類によって異なるかも。
![image.png (81.0 kB)](https://img.esa.io/uploads/production/attachments/3505/2017/01/03/10863/7047722e-3db0-4e8e-b379-5650d8841989.png)

## 放電状況を確認する
手元のeneloopを満充電にし実施したところ両方電池共に0.98Vまで下がったところでオートカットが機能している事がわかる。

![手元のeneloopを比較.jpg (305.5 kB)](https://img.esa.io/uploads/production/attachments/3505/2017/01/07/10863/b2e21705-78e5-4779-98c4-214d144c95c3.jpg)


# まとめ
放電機の作成までは順調だったが、そこからESP8266を利用してArduinoと通信を行いうまでが苦難の連続でした。
できる限りコストを削減し、モジュールも小型化したかったのでArduinoを選択したがメモリ不足だったり、I2C通信がうまくできなかったりと紆余曲折があり、非常に時間を掛けてしまったもののなんとかやりたい事が達成できて良かったです。
ESP8266単体ではポートが不足しがちで、アナログポートの利用も怪しいので、サブマイコンにArduinoを利用しようと考えたのが安易な発想でした。
今回はESP8266側とArduinoの電圧をそろえたかったので、Arduino Pro Miniを採用したが、１２Vバッテリーの検知だと５VのArduinoと分圧回路での測定になると思うので、降圧回路で、ESP8266を動作させた方が良いかもしれないです。

## 苦労したこと
* このスペースに全てを納めること
* マイコン部分を取り外し可能にしたりしているのでケースの高さギリギリに納める事が難しかった

![IMG_5654.JPG (1.3 MB)](https://img.esa.io/uploads/production/attachments/3505/2017/01/07/10863/cc75ca2a-1634-4b21-b259-ff4fea3ffa8e.JPG)



# 参考・関連
http://qiita.com/u_one/items/44b95274a40f9c40f4d0
http://tabrain.blog.so-net.ne.jp/2015-09-10
http://mixture-art.net/arduino%E3%81%A7i2c%E9%80%9A%E4%BF%A1%E3%82%92%E3%82%84%E3%82%8B%E9%9A%9B%E3%81%AE%E3%83%A1%E3%83%A2/
http://www.denshi.club/make/arduinoorg/wire.html
http://blog-yama.a-quest.com/?eid=970163
http://shirotsuku.sakura.ne.jp/blog/?p=710
https://tech-blog.cerevo.com/archives/859/
https://cdn.sparkfun.com/datasheets/Dev/Arduino/Boards/ProMini8MHzv1.pdf

