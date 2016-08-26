---
layout: post
title: NodeOSを起動してみた
published: true
---

村岡です。node.jsが好きなので[NodeOS](http://node-os.com/) がnode.jsベースのOSとか胸熱なのでとりあえず起動してみました。
Readmeをみるといろんな起動方法があるけど[dockerイメージが一番手軽そう](https://github.com/nodeos/nodeos#nodeos-on-lxc-containers-docker-and-vagga)なのでそれで。  
最初Macでやろうと思ったけどReadmeをみるかぎりではLinux系でやったほうが簡単そうだったのでUbuntu 16.04 LTS 64bitにdockerをインストールしてやってみた。

# 手順

## dockerのインストール

`wget -qO- https://get.docker.com/ | sh` でインストール。
`sudo usermod -aG docker $USER` でユーザーにdockerグループを追加

## NodeOSコンテナの起動

`sudo docker run -t -i nodeos/nodeos`

これだけ。

起動するとこうなる。

![スクリーンショット 2016-05-27 20.08.49.png (112.7 kB)](https://img.esa.io/uploads/production/attachments/3505/2016/05/27/10856/cd9c1b1f-7672-4953-af6a-a79d1da79191.png)

lsするとこうなる。

![スクリーンショット 2016-05-27 20.10.14.png (114.8 kB)](https://img.esa.io/uploads/production/attachments/3505/2016/05/27/10856/9b26638c-3ce1-45fb-8a85-86a90f7cc0f6.png)

カレントディレクトリのリストはJSの配列で返ってくるみたい。おもろい。
いま使えるコマンドは[こちら](https://github.com/NodeOS/NodeOS/wiki/Commands)。

## OSの停止

Ctrl+DでOSが停止する。node.jsのREPLっぽい。
いまんとこhaltコマンドとかは実装されてないみたい。

![スクリーンショット 2016-05-27 20.13.32.png (117.0 kB)](https://img.esa.io/uploads/production/attachments/3505/2016/05/27/10856/cbecb37a-1305-41d2-aa15-b3b47b59d7c8.png)

## 感想

テキストエディタとかどうなってるのかイマイチわからなかった。けっこう活発に開発がされているようなのでそのうちできてくるのかもしれないから今後にちょっと期待してます。
