---
title: CentOS7.6 で PX-W3PE4（第五回）録画コマンド導入編
description: CentOS7.6 で PX-W3PE4を動かしたい挑戦第五回です。
date: 2019-06-05
slug: centos76-px-w3pe4-5
tags:
  - Linux
  - PC-tv
series:
  - "CentOS7.6-PX-W3PE4"
---
## やりたいこと
CentOS 7.6環境で、PX-W3PE4を動かしたい。
[前回の記事](centos76-px-w3pe4-4)

## 環境

### PC

```bash
$ cat /etc/redhat-release
CentOS Linux release 7.6.1810 (Core)
```

NEC Mate PC-MK34MEZDG

### チューナー
[PLEX PX-W3PE4](http://www.plex-net.co.jp/product/px-w3pe4/)

### カードリーダー
[Identive "CLOUD 2700R, USB, white"](https://www.amazon.co.jp/gp/product/B00EUV2NVE/ref=ppx_yo_dt_b_asin_title_o05_s00?ie=UTF8&psc=1)

## やったこと

### arib25ライブラリの導入

```bash
$ cd ~/chinachu_work/
$ mkdir arib25
$ cd arib25/
$ wget http://hg.honeyplanet.jp/pt1/archive/c44e16dbb0e2.zip
$ unzip c44e16dbb0e2.zip
$ cd pt1-c44e16dbb0e2/
$ cd arib25/
$ make clean
$ make
$ sudo make install
```

### recpt1の導入

```bash
$ cd ~/chinachu_work
$ mkdir recpt1
$ cd recpt1
$ wget http://plex-net.co.jp/download/linux/Linux_Driver.zip
$ unzip Linux_Driver.zip
$ cd Linux_driver/MyRecpt1/MyRecpt1/recpt1
$ chmod +x ./autogen.sh
$ ./autogen.sh
$ chmod +x ./configure
$ ./configure --enable-b25
$ make
$ sudo make install
```

CentOSでは、/usr/local/libを自動では見に行ってくれないので、このままではエラーになります。

```bash
$ cd /etc/ld.so.conf.d
$ sudo vi custom.conf
```

ファイルの内容

```bash
/usr/local/lib
```

適用

```bash
$ sudo ldconfig
```

適用されているか確認

```bash
$ ldconfig -p | grep /usr/local/lib
        libarib25.so.0 (libc6,x86-64) => /usr/local/lib/libarib25.so.0
        libarib25.so (libc6,x86-64) => /usr/local/lib/libarib25.so
```

libarib25.so.0, libarib25.soの2ファイルがあればいいです。


### 実際に録画できるか確認

チャンネル番号を確認
マスプロのPDFを確認する。[channel.pdf](https://www.maspro.co.jp/contact/channel.pdf)

px4video0, px4video1はBS, CS用。px4video2, px4video3が地上波用です。

```bash
$ cd ~/chinachu_work
$ recpt1 -b --device /dev/px4video2 22 10 test1.ts

# デバイスファイルのパス チャンネル番号 録画秒数 出力ファイル名
```

出力されたtsファイルをSCPするとかして持ってきて確認します。正常に再生できたのでいいでしょう。

今回でこのシリーズはいったん終了にします。この後mirakurunとchinachuを入れますが、別シリーズとします。お疲れさまでした。
