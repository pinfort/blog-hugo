---
title: Raspberry Pi 4でおうちk8s構築（物理編）
description: raspberry Pi 4でおうちk8sを構築した記録です。
date: 2021-05-08
slug: raspberry-pi-4-k8s-1
tags:
  - Linux
  - k8s
  - network
categories:
  - "raspberry-pi-4-k8s"
---
[![raspberry pi 4](assets/thumbnail/respberrypi4-0-0-0-0-1620447724.jpg)](assets/respberrypi4.jpg)

## おうちk8s
今や、おうちにk8sクラスタがあるのは当たり前の時代。私もついにraspberry Pi 4でこれを構築することにしました。ラズパイでのk8sクラスタ構築例はインターネットにいくらでもあるので大変参考になりました。この記事では、特に物理的な構築部分について記述します。ソフトウェアの構築部分は別の記事で記述します。
[シリーズリンク](/categories/raspberry-pi-4-k8s)

## 材料
|名称|価格|購入場所|
|--|--|--|
|raspberry Pi 4 modelB 4GB|6,985*3|[KSY](https://raspberry-pi.ksyic.com/?pdp.id=498)|
|GeeekPi Raspberry Pi4クラスターケース|2,499|[Amazon](https://www.amazon.co.jp/gp/product/B07TJ15YL1)|
|tp-link TL-WR902AC ワイヤレストラベルルータ|3,160|[ビックカメラ](https://www.biccamera.com/bc/item/4694634/)|
|バッファロースイッチングハブ LSW6-GT-5EPL/BK|1,738|[ツクモ](https://shop.tsukumo.co.jp/goods/4981254050057)|
|適当なLANケーブル|110*3|[千石電商](https://www.sengoku.co.jp/mod/sgk_cart/detail.php?code=4ALA-J5LM#)|
|適当なUSB A-Cケーブル|528*3|千石電商|
|適当なUSB電源|2200|[千石電商](https://www.sengoku.co.jp/mod/sgk_cart/detail.php?code=EEHD-5KTX)|
|↑では足りないので家に落ちていたUSB電源|?|?|
|Sandisk microSDカード 32GB SDSQUA4-032G-GN6MN|510*3|[あきばお～](http://www.akibaoo.co.jp/c/item/0619659184162/)|

## ケース組み立て
とりあえずラズパイをケースにはめていきます。他の記事でも書かれているように、保護シートを外さないと白い板の状態ですので保護シートを外します。爪で何とかします。するときれいな透明になります。今回は下三段にラズパイ、四段目にトラベルルータ、屋上にスイッチングハブを配置しました。ハブは大きくてケースに入りませんでした。

いくつか迷ったところはありましたが、Amazonのケース商品ページの説明や、同梱されている説明を見ながらなんとか完成させました。微妙にねじが締まりきらないところもあって少し不安ですが大きな問題にはならないでしょう。ケースファンを刺すラズパイのピンの位置があっているか不安ですが、起動し見て動かなかったら見直しましょう。

スイッチングハブは両面テープで固定します。とはいえ、ケースにファン用の穴が開いているので、止めても強度がいまいちです。あまり触らないようにしましょう。トラベルルータは、この後初期設定で触るので後で止めることにします。

## ケーブリング
- 各ラズパイからスイッチングハブにLANケーブルをつなぎます。千石の店頭にあった一番短そうなケーブルを買ったのですが、やはり少し長いですね。多少余っていますが仕方ありません。
- スイッチングハブからトラベルルータにLANケーブルをつなぎます。なお、ここにはトラベルルータに同梱されていたLANケーブルを使いました。
- 各ラズパイにUSBケーブルを接続します。
- トラベルルータに同梱されている電源用USBケーブルを接続します。
- USB電源を用意し、各USBケーブルを電源につなぎます。

見た目上の組み立てはこんなものでしょうか。

{{< figure src="assets/thumbnail/respberrycluster.jpg" link="assets/respberrycluster.jpg" title="respberrycluster" height="340">}}

## おわりに
今回は、購入した資材を使ってクラスタの物理的な組み立てまでを行いました。3.5万円程でこれだけ構築できるのだからすごい時代です。ソフトウェアの構築はまだ一切行っていないために、今後修正が入る可能性もありますが、いったんこれでいいでしょう。次の記事で、いよいよk8sの構築をしていきたいと思います。
