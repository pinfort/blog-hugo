---
title: CentOS7.6 で PX-W3PE4（第四回）カードリーダー設定編
description: CentOS7.6 で PX-W3PE4を動かしたい挑戦第四回です。
date: 2019-06-05
slug: centos76-px-w3pe4-4
tags:
  - Linux
  - PC-tv
categories:
  - "CentOS7.6-PX-W3PE4"
---
## やりたいこと
CentOS 7.6環境で、PX-W3PE4を動かしたい。
[前回の記事](centos76-px-w3pe4-3)

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

今回はカードリーダーの設定をします。

```bash
$ sudo yum install ccid
...
完了しました！

$ sudo yum install pcsc-lite-devel
...
完了しました！

$ sudo yum install pcsc-tools
...
完了しました！

$ sudo systemctl enable pcscd
$ sudo systemctl start pcscd
$ pcsc_scan
...
Possibly identified card...
...
Japanese Chijou Digital B-CAS Card (pay TV)
```

今回のカードリーダーはすでにplistに登録されているようなので追加の作業は不要なようですね。B-CASカードが認識されましたね。カードリーダーの設定はこれでよさそうです。

[次回の記事](centos76-px-w3pe4-5)
