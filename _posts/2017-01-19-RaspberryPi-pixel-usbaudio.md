---
published: true
layout: post
title: RASPBIAN JESSIE WITH PIXEL でUSBオーディオアダプタを使う
tags:
  - rasberrypi
---

![image.png (159.4 kB)](https://img.esa.io/uploads/production/attachments/3505/2017/01/19/10863/a3b7ea87-216c-407b-9466-356e1afc36fe.png)


# はじめに
こんにちは。ナカニシです。
最近Raspberry Pi  に最新のOSの RASPBIAN JESSIE WITH PIXEL をインストールして遊んでいますが、まず見た目がcoolになって良いですね。テンションあがります。

さて今回は思いつきで、RASPBIAN JESSIE WITH PIXEL でUSBオーディオアダプタを認識させてみました。

## 動機
Raspberry Pi 3 に標準装備されている、ミニジャックにイヤホンを繋いで、試しにaplayコマンドでwavファイルを再生させたのですが･･･
すごいノイズが入ってまともに聞けたものでは無かったです(>_<)
ならばサウンドカード側で出力させればマシになるのでは？と思いつき実際に試してみました。

※ノイズの原因は私の勘違いからなるもので、まとめに記載しております。


## 対象読者
* RaspberryPi 3 B で音声ファイルを再生させたい方

## 検証環境
* RaspberryPi 3 B
* OS: RASPBIAN JESSIE WITH PIXEL Release date:2017-01-11 Kernel version:4.4
* [Amazon \| Sabrent USB オーディオ変換アダプタ](https://www.amazon.co.jp/Sabrent-AU-MMSA-USB-%E3%82%AA%E3%83%BC%E3%83%87%E3%82%A3%E3%82%AA%E5%A4%89%E6%8F%9B%E3%82%A2%E3%83%80%E3%83%97%E3%82%BF-Windows%E3%81%A8Mac%E3%81%AB%E5%AF%BE%E5%BF%9C-%E3%83%89%E3%83%A9%E3%82%A4%E3%83%90%E3%83%BC%E4%B8%8D%E8%A6%81/dp/B00IRVQ0F8/ref=pd_sim_147_1?_encoding=UTF8&psc=1&refRID=RPCQB9ZC0G0GK24KEW47)

# まずは、オーディオミニジャックより再生させてみる
```
$ aplay /usr/share/sounds/alsa/Front_Center.wav 
Playing WAVE '/usr/share/sounds/alsa/Front_Center.wav' : Signed 16 bit Little Endian, Rate 48000 Hz, Mono
```
3秒間ほどのサンプルファイルが再生されました。
バチバチの耳につくノイズが混ざって聞こえてきました。


# USBオーディオアダプタを利用する
USBポートに差し込み、USBオーディオアダプタのスピーカー端子側に接続して、コマンドにて再生させてみました･･･何も聞こえません。

コマンドにて確認すると、認識はされているようです。
ドライバを入れる必要が無いのは素敵です。

```
$ sudo lsusb
Bus 001 Device 005: ID 0d8c:0014 C-Media Electronics, Inc.  ←USBオーディオアダプタ
Bus 001 Device 003: ID 0424:ec00 Standard Microsystems Corp. SMSC9512/9514 Fast Ethernet Adapter
Bus 001 Device 002: ID 0424:9514 Standard Microsystems Corp. 
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```
結果としてはオーディオミニジャックより再生されていました。
調べてみると、USBオーディオアダプタを優先して認識するように設定が必要との事で早速やってみます。


## オーディオアダプタの優先順位を変更する
先に出力先を変更します。

```
１． $ sudo raspi-config を実行し、設定メニューを表示させます
２．メニューより （8 Advanced Options） を選択します
３．更に（A9 Audio）を選択します
４．音声出力先の設定になるので （Auto）を選択しました。
　AutoだとHDMI→HeadPhoneの順で接続を確認し出力を行うそうです。
```

次に設定ファイルを変更します。

```
$ sudo vi /usr/share/alsa/alsa.conf
```

ファイル内の以下の箇所を修正しました。

```
〜〜〜

@hooks [
   {
      func load
      files [
         {
            @func concat
            strings [
               { @func datadir }
               "/alsa.conf.d/"
            ]
         }
         "/etc/asound.conf"
#         "~/.asoundrc"         ←コメントアウトする
      ]
      errors false
   }
]

〜〜〜
defaults.ctl.card 1         ←０を１に変更
defaults.pcm.card 1         ←０を１に変更
〜〜〜
```

編集後は保存し、再起動します。

```
$ sudo reboot
```

## テスト再生してみる
再起動後に再度テスト再生してみます。

```
$ aplay /usr/share/sounds/alsa/Front_Center.wav 
```
ノイズ混じりだったのがクリアに再生されました。

# マイクも利用してみる
音声が無事に出力できたので、USBオーディオアダプタのマイク側も利用できるか確認してみます。

まずデバイスのIDを調べてみます。card 1と認識されていました。

```
$ arecord -l
**** List of CAPTURE Hardware Devices ****
card 1: Device [USB Audio Device], device 0: USB Audio [USB Audio]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
```

デバイスの現在の設定値を調べてみます。
マイクは0-35の範囲で設定でき、現在20に設定されていました。

```
$ amixer sget Mic -c 1
Simple mixer control 'Mic',0
  Capabilities: pvolume pvolume-joined cvolume cvolume-joined pswitch pswitch-joined cswitch cswitch-joined
  Playback channels: Mono
  Capture channels: Mono
  Limits: Playback 0 - 31 Capture 0 - 35
  Mono: Playback 16 [52%] [-7.00dB] [off] Capture 20 [57%] [8.00dB] [on]
```

入力レベルを変更してみます。

```
 $mixer sset Mic 35 -c 1
Simple mixer control 'Mic',0
  Capabilities: pvolume pvolume-joined cvolume cvolume-joined pswitch pswitch-joined cswitch cswitch-joined
  Playback channels: Mono
  Capture channels: Mono
  Limits: Playback 0 - 31 Capture 0 - 35
  Mono: Playback 31 [100%] [8.00dB] [off] Capture 35 [100%] [23.00dB] [on]
```

マイクのテストをしてみます。
マイクから入力された音声が、スピーカーより出力されました。
停止は ctl+c キーで停止出来ます。

```
 $ arecord -f S16_LE -r 44100 -D hw:1 | aplay
Recording WAVE 'stdin' : Signed 16 bit Little Endian, Rate 44100 Hz, Mono
Playing WAVE 'stdin' : Signed 16 bit Little Endian, Rate 44100 Hz, Mono
Aborted by signal Interrupt...
```

# まとめ
ここまでやってみて、ノイズの原因が見えてきたのですが、RaspberryPi 3 の3.5mmのオーディオジャックとコンポジット端子のようです。
本家のサイト[ラズベリーパイ3モデルB ](https://www.raspberrypi.org/products/raspberry-pi-3-model-b/) にもきっちり

```
Combined 3.5mm audio jack and composite video
```

と記載がありました。

ノイズの原因は、RaspberryPi Bのオーディオ出力ジャックは通常の３極（L,R,G）ではなく、ビデオ出力も含んだ４極（L,R,G,V）であるために、接続したイヤホンが原因でノイズが発生していたようです。



RaspberryPiはHDMI出力も備えていますので、TVより出力しても良いかと思うのですが、マシンパワーが非力なためにデジタル変換に時間がかかり音声が送れたり、漏れたりと思わぬ不具合が発生するという報告もされているようですので、サウンドカードでの出力も選択肢としては正しかったのかと思いました。




# 参考・関連
[オーディオも自作すると面白い！: piCorePlayer：RaspberryPi B\+のオーディオ出力がノイズだらけ。。。\(T\_T\)](http://kitatokyo2013.blogspot.jp/2015/02/picoreplayerraspberrypi-b.html)
[Raspberry Pi \| aplayコマンドで音声の冒頭が再生されない問題→アナログスピーカーで解決！ – たぷん日記](http://www.tapun.net/raspi/raspberry-pi-analog-speaker)
