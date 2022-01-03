---
title: CentOS7.6 で PX-W3PE4（第一回）公式ドライバチャレンジ編
description: CentOS 7.6環境でPX-W3PE4を動かす挑戦第一回です。
date: 2019-06-01
slug: centos76-px-w3pe4-1
tags:
  - Linux
  - PC-tv
categories:
  - "CentOS7.6-PX-W3PE4"
---
## やりたいこと

CentOS7.6 環境で録画鯖を作りたい。

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

### 参考
[Linuxで録画鯖を建てる #03「CentOS7.5 + Mirakurun + Chinachu」](https://yy-kuso.hatenablog.com/entry/recorder03)

## やったこと

### 作業用ディレクトリの作成

```bash
$ mkdir ~/chinachu_work
$ cd ~/chinachu_work
```

### チューナードライバの導入

PlexのサポートページよりLinux用ドライバをダウンロード。CentOS7-1804\_64bit_kernel3.10.0-862を選んでみる。

```bash
$ wget http://plex-net.co.jp/plex/linux/CentOS7-1804_64bit_kernel3.10.0-862.zip
--2019-06-01 18:58:52--  http://plex-net.co.jp/plex/linux/CentOS7-1804_64bit_kernel3.10.0-862.zip
plex-net.co.jp (plex-net.co.jp) をDNSに問いあわせています... 157.7.144.5
plex-net.co.jp (plex-net.co.jp)|157.7.144.5|:80 に接続しています... 接続しました。
HTTP による接続要求を送信しました、応答を待っています... 200 OK
長さ: 2402958 (2.3M) [application/zip]
`CentOS7-1804_64bit_kernel3.10.0-862.zip' に保存中

100%[======================================>] 2,402,958   1.52MB/s 時間 1.5s

2019-06-01 18:58:53 (1.52 MB/s) - `CentOS7-1804_64bit_kernel3.10.0-862.zip' へ保存完了 [2402958/2402958]
```
```bash
$ unzip CentOS7-1804_64bit_kernel3.10.0-862.zip
Archive:  CentOS7-1804_64bit_kernel3.10.0-862.zip
  inflating: CentOS7-1804_64bit_kernel3.10.0-862/tty_Virtual.ko
  inflating: CentOS7-1804_64bit_kernel3.10.0-862/usb-px4.ko
  
$ cd CentOS7-1804_64bit_kernel3.10.0-862
$ sudo insmod tty_Virtual.ko
$ sudo insmod usb-px4.ko
insmod: ERROR: could not insert module usb-px4.ko: Invalid parameters
```
は？エラーになりましたね。というのも、このドライバはCentOS7.5用。このPCのCentOSは7.6です。こないだ上げちゃいました。何とか動かせないでしょうか。

```bash
$ dmesg | tail
[ 1470.374192] usb_px4: disagrees about version of symbol usb_bulk_msg
[ 1470.374193] usb_px4: Unknown symbol usb_bulk_msg (err -22)
[ 1470.374197] usb_px4: disagrees about version of symbol usb_get_dev
[ 1470.374198] usb_px4: Unknown symbol usb_get_dev (err -22)
[ 1470.374200] usb_px4: disagrees about version of symbol usb_submit_urb
[ 1470.374201] usb_px4: Unknown symbol usb_submit_urb (err -22)
[ 1470.374214] usb_px4: disagrees about version of symbol usb_unlink_urb
[ 1470.374215] usb_px4: Unknown symbol usb_unlink_urb (err -22)
[ 1470.374220] usb_px4: disagrees about version of symbol usb_kill_urb
[ 1470.374221] usb_px4: Unknown symbol usb_kill_urb (err -22)
```

ググったら以下のようなサイトがヒットしました。
[Linuxのカーネルモジュールの整合性検証の仕組み](https://blog.bitmeister.jp/?p=2916)
なるほど。ドライバはカーネルモジュールで、カーネルモジュールの情報はmodinfoというコマンドで見ることができると。

```bash
$ modinfo usb-px4.ko
filename:       /home/pinfort/chinachu_work/CentOS7-1804_64bit_kernel3.10.0-862/usb-px4.ko
license:        GPL
version:        v18.06.04.1
description:    DTV Native Driver for Devices Based on ITEtech it930x
author:         Jacky Han <jackysummerhan@msn.com>
retpoline:      Y
rhelversion:    7.5
srcversion:     30CED62D4B8E4D4F7DFAE22
alias:          usb:v0511p024Ad*dc*dsc*dp*ic*isc*ip*in*
alias:          usb:v0511p023Fd*dc*dsc*dp*ic*isc*ip*in*
alias:          usb:v0511p084Ad*dc*dsc*dp*ic*isc*ip*in*
alias:          usb:v0511p083Fd*dc*dsc*dp*ic*isc*ip*in*
depends:        tty_Virtual
vermagic:       3.10.0-862.3.3.el7.x86_64 SMP mod_unload modversions
parm:           debug:int
parm:           URB_BUFSIZE:URB_BUFSIZE = 188*n (int)
```
確かに7.5にターゲットされているようです。

insmodコマンドを使っていましたが、modprobeコマンドという、これを代替するコマンドがあり、これにはforceオプションがあるようです。これを使ってみましょう。

```bash
$ sudo rmmod tty_Virtual.ko
```

```bash
$ sudo modprobe tty_Virtual.ko
modprobe: FATAL: Module tty_Virtual.ko not found.
```

うまくいきません。
そこで、こんなページを見つけました。

[[Linux][kernel] カーネルモジュールの情報を確認する方法まとめ](https://qiita.com/koara-local/items/309aec36352f113d4519)

```bash
# モジュールのロード（依存関係を考慮する）
# /lib/modules/$(uname -r)/kernel/drivers 以下から <module_name> を読み込み
$ modprobe <module_name>
```

とあります。/lib/modules/$(uname -r)/kernel/driversに移動すればよさそうですね。

```bash
$ ls /lib/modules
3.10.0-693.el7.x86_64       3.10.0-957.12.2.el7.x86_64
3.10.0-862.14.4.el7.x86_64  3.10.0-957.el7.x86_64

$ uname -r
3.10.0-957.12.2.el7.x86_64

$ sudo mkdir /lib/modules/3.10.0-957.12.2.el7.x86_64/kernel/drivers/tvtuner
$ sudo cp tty_Virtual.ko /lib/modules/3.10.0-957.12.2.el7.x86_64/kernel/drivers/tvtuner/
$ sudo cp usb-px4.ko /lib/modules/3.10.0-957.12.2.el7.x86_64/kernel/drivers/tvtuner/
$ ls /lib/modules/3.10.0-957.12.2.el7.x86_64/kernel/drivers/tvtuner/
tty_Virtual.ko  usb-px4.ko

# このままではまだ module not foundです。

# modulesを再検索してキャッシュ更新？的なことをします
$ sudo depmod -a

$ sudo modprobe tty_Virtual
$ lsmod | grep tty
tty_Virtual            22223  0

# tty_Virtualが入りました！

$ sudo modprobe usb_px4
modprobe: ERROR: could not insert 'usb_px4': Invalid argument

# やはりusb_px4.koの方でエラーが出るようですね

$ sudo modprobe usb_px4 -f
# force オプションで入れてみます

$ lsmod | grep px4
usb_px4               468143  0
tty_Virtual            22223  1 usb_px4
```

何とかなるでしょうか？

```bash
$ ls /dev/px4*
ls: /dev/px4* にアクセスできません: そのようなファイルやディレクトリはありません
```
再起動してないので意味ないんですが、まあやめといたほうがいい気しかしないので公式ドライバのインストールをあきらめて、次回、第二回で非公式ドライバのインストールを試みます。

[次回の記事](centos76-px-w3pe4-2)
