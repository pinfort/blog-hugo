---
title: CentOS7.6 で PX-W3PE4（第三回）不具合調査編
description: CentOS7.6 で PX-W3PE4を動かしたい挑戦第三回です。
date: 2019-06-02
slug: centos76-px-w3pe4-3
tags:
  - Linux
  - PC-tv
series:
  - "CentOS7.6-PX-W3PE4"
---
## やりたいこと
CentOS 7.6環境で、PX-W3PE4を動かしたい。
[前回の記事](/posts/centos76-px-w3pe4-2)

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

以前dkmsで非公式ドライバを入れたとき、エラーになっていたのを思い出したので、念のため入れなおしておきます

```bash
$ sudo dkms uninstall -m px4_drv/0.2.1

-------- Uninstall Beginning --------
Module:  px4_drv
Version: 0.2.1
Kernel:  3.10.0-957.12.2.el7.x86_64 (x86_64)
-------------------------------------

Status: Before uninstall, this module version was ACTIVE on this kernel.
Removing any linked weak-modules

px4_drv.ko.xz:
 - Uninstallation
   - Deleting from: /lib/modules/3.10.0-957.12.2.el7.x86_64/extra/
 - Original module
   - No original module was found for this module on this kernel.
   - Use the dkms install command to reinstall any previous module version.


Running the post_remove script:
`/etc/udev/rules.d/99-px4video.rules' を削除しました
depmod...

DKMS: uninstall completed.

$ sudo dkms remove px4_drv/0.2.1 --all

-------- Uninstall Beginning --------
Module:  px4_drv
Version: 0.2.1
Kernel:  3.10.0-957.12.2.el7.x86_64 (x86_64)
-------------------------------------

Status: Before uninstall, this module version was ACTIVE on this kernel.
Removing any linked weak-modules

px4_drv.ko.xz:
 - Uninstallation
   - Deleting from: /lib/modules/3.10.0-957.12.2.el7.x86_64/extra/
 - Original module
   - No original module was found for this module on this kernel.
   - Use the dkms install command to reinstall any previous module version.


Running the post_remove script:
`/etc/udev/rules.d/99-px4video.rules' を削除しました
depmod...

DKMS: uninstall completed.

------------------------------
Deleting module version: 0.2.1
completely from the DKMS tree.
------------------------------
Done.

```

再インストール。

```bash

$ cd ~/chinachu_work/px4_drv

$ sudo bash dkms.install

Creating symlink /var/lib/dkms/px4_drv/0.2.1/source ->
                 /usr/src/px4_drv-0.2.1

DKMS: add completed.

Kernel preparation unnecessary for this kernel.  Skipping...

Building module:
cleaning build area...
cd ./driver; make KVER=3.10.0-957.12.2.el7.x86_64 px4_drv.ko...
cleaning build area...

DKMS: build completed.

px4_drv.ko.xz:
Running module version sanity check.
 - Original module
   - No original module exists within this kernel
 - Installation
   - Installing to /lib/modules/3.10.0-957.12.2.el7.x86_64/extra/
Adding any weak-modules

Running the post_install script:
`./etc/99-px4video.rules' -> `/etc/udev/rules.d/99-px4video.rules'

depmod...

DKMS: install completed.

$ sudo reboot
```
再起動しておきます。

```bash
$ lsusb
Bus 002 Device 002: ID 8087:8000 Intel Corp.
Bus 002 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 001 Device 002: ID 8087:8008 Intel Corp.
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 004 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 003 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```

だめですね。

さて、ここまでで考えられるのは、

1. チューナー本体の不具合
2. チューナーのUSB2.0端子とUSB3.0端子を変換するために使ったコードの不具合（Linux機の内部にUSB2.0の口がなかったため）
3. PC側のUSB端子の不具合
4. PC側のPCI端子の不具合

でしょうか。一つずつ潰していきましょうか。

1. チューナー本体の不具合

    自宅にあるWindows機にこのチューナーを刺したところ、正常に認識されました。
2. USB2.0 to USB3.0変換コードの不具合

    Windows機にこのコードを利用して接続しても、正常に認識されました。
3. Linux機のUSB端子の不具合

    このUSB端子にもともとついていたUSB-Aに変換するコードを接続して、その辺にあったUSBデバイスを接続したところ、Linux機でも正常に認識しました。
4. PC側のPCI端子の不具合

    このPCにはPCI端子が三つあるのですが、どれに刺してもlsusbに出てきません。ls /devにも出てきません。
    外に手ごろなPCIカードがあればいいのですがあいにくないので困りました。
    
    検証用に、USB拡張ボードを買ってきました。どうやら正常に認識されているようです。
    
さて。あと可能性は何だろう？

ここで、再起動時のdmesgを見てみたところ、以下のような記述があった。

```bash
[    0.677352] px4_drv: loading out-of-tree module taints kernel.
[    0.677686] px4_drv: module verification failed: signature and/or required key missing - tainting kernel
[    0.678991] px4_drv: px4_drv version 0.2.1, rev: 141, commit: 2b3f79b5bc5db56e8556bb28397f7d8f74b2adeb @ master
[    0.679007] usbcore: registered new interface driver px4_drv
```

うーん...これはロードされているのか？

### 解決

この後内部USBからtypeAに変換するコードを買ってきて、PC外部のtypeAに接続したところ、無事正常に認識しました。要するに違うUSBポートならいけたということですね。物さえ買ってくればあっさりでしたね。今は外部にコードが回っていて気持ち悪いので、そのうち内部に回します。

```bash
$ ls /dev | grep px4
px4video0
px4video1
px4video2
px4video3
```

やっとデバイスファイルが見えました。ということで、次回はカードリーダーの設定をしていきます。

[次回の記事](/posts/centos76-px-w3pe4-4)
