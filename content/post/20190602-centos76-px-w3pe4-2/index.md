---
title: CentOS7.6 で PX-W3PE4（第二回）非公式ドライバチャレンジ編
description: CentOS7.6環境でPX-W3PE4を動かす挑戦第二回です。
date: 2019-06-02
slug: centos76-px-w3pe4-2
tags:
  - Linux
  - PC-tv
categories:
  - "CentOS7.6-PX-W3PE4"
---
## やりたいこと
CentOS 7.6環境で、PX-W3PE4を動かしたい。
[前回の記事](centos76-px-w3pe4-1)

## 環境

### PC

```bash
$ cat /etc/redhat-release
CentOS Linux release 7.6.1810 (Core)
```

### チューナー
[PLEX PX-W3PE4](http://www.plex-net.co.jp/product/px-w3pe4/)

### カードリーダー
[Identive "CLOUD 2700R, USB, white"](https://www.amazon.co.jp/gp/product/B00EUV2NVE/ref=ppx_yo_dt_b_asin_title_o05_s00?ie=UTF8&psc=1)

## やったこと

以前公式ドライバを無理やり入れたので、とりあえず消しておきます。

```bash
$ sudo modprobe usb_px4 -fr
$ sudo modprobe tty_Virtual -r
$ lsmod | grep px4
$ cd ~/chinachu_work
$ rm CentOS7-1804_64bit_kernel3.10.0-862.zip
$ rm -rf CentOS7-1804_64bit_kernel3.10.0-862/
```

公式ドライバがなくなりました。非公式入れていくぞ！

[PLEX社製TVチューナーの非公式Linuxドライバインストール方法](https://www.jifu-labo.net/2019/01/unofficial_plex_driver/)
この記事を見ながら進めていきます。

非公式ドライバはオープンソースで開発されていて、GitHubにあります。

[nns779/px4_drv](https://github.com/nns779/px4_drv)

> PLEX PX-W3U4/Q3U4/W3PE4/Q3PE4用の非公式版Linuxドライバです。
PLEX社のWebサイトにて配布されている公式Linuxドライバとは別物です。

これを導入していきます。
とりあえずドライバのソースコードの入手。

```bash
$ cd ~/chinachu_work
$ git clone https://github.com/nns779/px4_drv.git
```

次に、Windows版ドライバの入手。

```bash
$ wget http://plex-net.co.jp/download/pxw3pe4v1.4.zip
--2019-06-01 20:41:12--  http://plex-net.co.jp/download/pxw3pe4v1.4.zip
plex-net.co.jp (plex-net.co.jp) をDNSに問いあわせています... 157.7.144.5
plex-net.co.jp (plex-net.co.jp)|157.7.144.5|:80 に接続しています... 接続しました。
HTTP による接続要求を送信しました、応答を待っています... 200 OK
長さ: 322891 (315K) [application/zip]
`pxw3pe4v1.4.zip' に保存中

100%[======================================>] 322,891     1.01MB/s 時間 0.3s

2019-06-01 20:41:12 (1.01 MB/s) - `pxw3pe4v1.4.zip' へ保存完了 [322891/322891
```

ファームウエア抽出ツールのビルド。

```bash
$ cd px4_drv/fwtool/
$ make
gcc -O2 -Wall   -c -o fwtool.o fwtool.c
gcc -O2 -Wall   -c -o tsv.o tsv.c
gcc -O2 -Wall   -c -o crc32.o crc32.c
gcc -o fwtool fwtool.o tsv.o crc32.o
```

ファームウエアの抽出。

```bash
$ cd ~/chinachu_work
$ unzip pxw3pe4v1.4.zip
Archive:  pxw3pe4v1.4.zip
  inflating: PX-W3PE4_Driver_Manual.pdf
  inflating: x64/pxw3pe4.cat
  inflating: x64/PXW3PE4.inf
  inflating: x64/PXW3PE4.sys
  inflating: x86/pxw3pe4.cat
  inflating: x86/PXW3PE4.inf
  inflating: x86/PXW3PE4.sys

$ ./px4_drv/fwtool/fwtool -h
fwtool for px4 drivers

usage: fwtool <driver binary> <output>

$ ls
PX-W3PE4_Driver_Manual.pdf  px4_drv  pxw3pe4v1.4.zip  x64  x86

# unzipをミスってファイルが散らかってしまいましたがまあいいでしょう。

$ sudo ./px4_drv/fwtool/fwtool x64/PXW3PE4.sys it930x-firmware.bin
fwtool for px4 drivers

Driver file: x64/PXW3PE4.sys
Output file: it930x-firmware.bin

Couldn't open file 'fwinfo.tsv' to read.
Failed to load firmware information file.

# fwinfo.tsvが読めないといっていますね。fwinfo.tsvはちゃんとfwtoolと同じディレクトリにあるので、
# おそらく内部で同じディレクトリにあるとして読み込んでいるのでしょうが、
# 実行ディレクトリを探してしまって見つからないとなっているのでしょう。
# ということで、fwtoolのあるディレクトリから実行します。

$ cd px4_drv/fwtool/
$ sudo ./fwtool ~/chinachu_work/x64/PXW3PE4.sys it930x-firmw
are.bin
fwtool for px4 drivers

Driver file: /home/pinfort/chinachu_work/x64/PXW3PE4.sys
Output file: it930x-firmware.bin

Driver description: PX-W3PE4 BDA Ver.1.4 64bit
Firmware length: 7013 bytes
Firmware CRC32: df0bf49a
OK.

$ ls
Makefile      crc32.c  crc32.o     fwtool    fwtool.o             tsv.c  tsv.o
Makefile.dep  crc32.h  fwinfo.tsv  fwtool.c  it930x-firmware.bin  tsv.h
```

it930x-firmware.binができましたね。
移動します

```bash
$ sudo cp -p it930x-firmware.bin /lib/firmware/
$ ls /lib/firmware/ | grep it930
it930x-firmware.bin
```

ドライバのビルド用に、dkmsを入れます。
[CentOS7 DKMSのインストール](http://note.websmil.com/linux/centos/centos7-dkmsのインストール)

```bash
$ sudo yum install dkms
...
完了しました!
````

epel-releaseが入ってればすんなり入ってくれそうですね

dkmsを使ったドライバのビルド

```bash
$ cd ~/chinachu_work/px4_drv
$ vi dkms.install
```

dkms.installに書きます

```bash
. ./dkms.conf
cp -a `pwd` /usr/src/$PACKAGE_NAME-$PACKAGE_VERSION
dkms add -m $PACKAGE_NAME -v $PACKAGE_VERSION
dkms build -m $PACKAGE_NAME -v $PACKAGE_VERSION
dkms install -m $PACKAGE_NAME -v $PACKAGE_VERSION
```

実行します

```bash
$ sudo bash dkms.install

Creating symlink /var/lib/dkms/px4_drv/0.2.1/source ->
                 /usr/src/px4_drv-0.2.1

DKMS: add completed.
Error! echo
Your kernel headers for kernel 3.10.0-957.12.2.el7.x86_64 cannot be found at
/lib/modules/3.10.0-957.12.2.el7.x86_64/build or /lib/modules/3.10.0-957.12.2.el7.x86_64/source.
Error! echo
Your kernel headers for kernel 3.10.0-957.12.2.el7.x86_64 cannot be found at
/lib/modules/3.10.0-957.12.2.el7.x86_64/build or /lib/modules/3.10.0-957.12.2.el7.x86_64/source.
```

あら、エラーになってしまいました。
kernel-develを入れば良さそうです。

```bash
$ sudo yum install kernel-devel
...
完了しました!

# 再チャレンジ

$ sudo bash dkms.install
Error! DKMS tree already contains: px4_drv-0.2.1
You cannot add the same module/version combo more than once.

Kernel preparation unnecessary for this kernel.  Skipping...
...
DKMS: install completed.
$ sudo reboot
```
最初にエラーになってますが、二回目の実行だから重複でエラーになってるだけなのでまあ大丈夫でしょう。ともかくできたみたいです。

```bash
$ ls -l /dev/px4*
ls: /dev/px4* にアクセスできません: そのようなファイルやディレクトリはありません
```

あれぇ？起動時のログを見てみます。

```bash
$ sudo cat /var/log/messages | grep usb_px4
Jun  1 19:06:57 vpn kernel: usb_px4: disagrees about version of symbol usb_alloc_urb
Jun  1 19:06:57 vpn kernel: usb_px4: Unknown symbol usb_alloc_urb (err -22)
Jun  1 19:06:57 vpn kernel: usb_px4: disagrees about version of symbol usb_free_urb
Jun  1 19:06:57 vpn kernel: usb_px4: Unknown symbol usb_free_urb (err -22)
Jun  1 19:06:57 vpn kernel: usb_px4: disagrees about version of symbol usb_bulk_msg
Jun  1 19:06:57 vpn kernel: usb_px4: Unknown symbol usb_bulk_msg (err -22)
Jun  1 19:06:57 vpn kernel: usb_px4: disagrees about version of symbol usb_get_dev
Jun  1 19:06:57 vpn kernel: usb_px4: Unknown symbol usb_get_dev (err -22)
Jun  1 19:06:57 vpn kernel: usb_px4: disagrees about version of symbol usb_submit_urb
Jun  1 19:06:57 vpn kernel: usb_px4: Unknown symbol usb_submit_urb (err -22)
Jun  1 19:06:57 vpn kernel: usb_px4: disagrees about version of symbol usb_unlink_urb
```

やたらエラーになってますね。なんででしょう？

```bash
$ ls /lib/modules/3.10.0-957.12.2.el7.x86_64/kernel/drivers/tvtuner
tty_Virtual.ko  usb-px4.ko
```

公式のドライバを消してませんね。消して再起動してみましょう。

```bash
$ sudo rm -rf /lib/modules/3.10.0-957.12.2.el7.x86_64/kerne
l/drivers/tvtuner
$ sudo reboot
```

```bash
$ ls -l /dev/px4*
ls: /dev/px4* にアクセスできません: そのようなファイルやディレクトリはありません
```

むりですね...
そんなとき、以下のページを見つけました。
[pt3ドライバのインストール(debianのdkmsで野良カーネルモジュールをインストール)](https://haruo31.underthetree.jp/2013/08/28/pt3ドライバのインストールdebianのdkmsで野良カーネルモ/)

そうか。dkms.installした後もmodprobeしないといけないのか。

```bash
$ ls /lib/modules/3.10.0-957.12.2.el7.x86_64/extra
px4_drv.ko.xz

# ここにinstallされているみたいですね
# modprobeしてみましょう

$ sudo modinfo px4_drv
filename:       /lib/modules/3.10.0-957.12.2.el7.x86_64/extra/px4_drv.ko.xz
firmware:       it930x-firmware.bin
license:        GPL v2
description:    Unofficial Linux driver for PLEX PX-W3U4/Q3U4/W3PE4/Q3PE4 ISDB-T/S receivers
author:         nns779
version:        0.2.1
retpoline:      Y
rhelversion:    7.6
srcversion:     08980F3F6481F97E9CF34CD
alias:          usb:v0511p024Ad*dc*dsc*dp*ic*isc*ip*in*
alias:          usb:v0511p023Fd*dc*dsc*dp*ic*isc*ip*in*
alias:          usb:v0511p084Ad*dc*dsc*dp*ic*isc*ip*in*
alias:          usb:v0511p083Fd*dc*dsc*dp*ic*isc*ip*in*
depends:
vermagic:       3.10.0-957.12.2.el7.x86_64 SMP mod_unload modversions
parm:           xfer_packets:Number of transfer packets from the device. (default: 816) (uint)
parm:           urb_max_packets:Maximum number of TS packets per URB. (default: 816) (uint)
parm:           max_urbs:Maximum number of URBs. (default: 6) (uint)
parm:           tsdev_max_packets:Maximum number of TS packets buffering in tsdev. (default: 2048) (uint)
parm:           psb_purge_timeout:int
parm:           no_dma:bool
parm:           disable_multi_device_power_control:bool
parm:           s_agc_negative_mode:bool
parm:           s_vga_atten:bool
parm:           s_fine_gain:uint

# ちゃんとCentOS7.6向けにターゲットされています。

$ sudo modprobe px4_drv
$ lsmod | grep px4
px4_drv                85703  0
```

入りました。

```bash
$ sudo reboot
```

いけるかな？

```bash
$ ls /dev | grep px4
```

だめですね。

ここで、modproveが再起動時に自動で行われていないことに気づきました。ということで、再起動時にモジュールをロードするよう設定します。

(カーネルモジュールの制御)[https://vinelinux.org/docs/vine6/cui-guide/kernel-modules.html]を参考にしてみます。

```bash
$ sudo vi /etc/modules-load.d/px4_drv.conf
```

ファイルの内容
```bash
px4_drv
```

```bash
$ sudo reboot
```

```bash
$ lsmod | grep px
px4_drv                85703  0
```

やったぁ。しかし。

```bash
$ ls /dev/ | grep px
```
ない。

```bash
$ sudo yum install usbutils
$ lsusb
Bus 002 Device 002: ID 8087:8000 Intel Corp.
Bus 002 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 001 Device 002: ID 8087:8008 Intel Corp.
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 004 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 003 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```

そもそも認識されてなさそうだ。これは何か物理的な不具合を疑わなければならなさそうですね。

次回、不具合に関する調査をしていきます。

[次回の記事](centos76-px-w3pe4-3)
