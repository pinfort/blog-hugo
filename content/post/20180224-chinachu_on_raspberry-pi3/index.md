---
title: raspberry pi3でchinachu
description: raspberry pi 3でchinachuを使う。
date: 2018-02-24
slug: chinachu_on_raspberry-pi3
tags:
  - "chinachu"
  - "Linux"
  - "raspberrypi"
---
### 基本的にはここを参考に
- [https://qiita.com/shotasano/items/3809b8f3e0b62d51d3c3](https://qiita.com/shotasano/items/3809b8f3e0b62d51d3c3)
- [http://k-pi.hatenablog.com/entry/2017/07/30/151659](http://k-pi.hatenablog.com/entry/2017/07/30/151659)
- [https://qiita.com/ww24/items/0adc36c013511524da80](https://qiita.com/ww24/items/0adc36c013511524da80)

## 注意すること
- plistに書き加えないとカードリーダーを認識しないこと
- エラーが出たらsudoでやり直してみる。
- 再起動してみる。

## 困ったこと
- chinachu installerがいつまでも終わらない
->我慢して待とう
- チューナーがうまく使えない
->なんか再起動したら行けた
- 番組表のチャンネルが一部表示されない
->取得中。しばらくまとう
- ライブ視聴ができない
  - （追記）結論から言うと、ラズパイではライブ視聴できませんでした。ラズパイのスペックだと厳しいのかも。
