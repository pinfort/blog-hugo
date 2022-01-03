---
title: CentOS 8.1でDD Max M4
description: CentOS 8.1でDD Max M4
date: 2020-03-28
slug: centos81-dd-max-m4-1
tags:
  - CentOS
  - Linux
  - PC-tv
categories:
  - "CentOS8.1-DD-Max-M4"
---
## 環境

### PC

中古のDiginnos Raytrek LT K600

```
$ cat /etc/redhat-release
CentOS Linux release 8.1.1911 (Core)
```

### チューナー

[DD Max M4](https://www.digital-devices.eu/shop/en/tv-cards/tv-cards-for-pcie/341/4x-multi-band-tuner-tv-card-dd-max-m4)

### カードリーダー

[ACR39-NTTCom](https://www.ntt.com/business/services/application/authentication/jpki/download6.html)

## やったこと

### ドライバを入れる
インストール
```
$ sudo dnf update
$ sudo dnf install mercurial perl-Proc-ProcessTable kernel-devel kernel-headers
$ sudo dnf install @Standard '@Development Tools' -x cockpit*
$ sudo dnf install elfutils-libelf-devel
$ mkdir recwd
$ cd recwd
$ sudo wget https://github.com/DigitalDevices/dddvb/archive/0.9.37.tar.gz
$ sudo tar -xf 0.9.37.tar.gz
$ cd dddvb-0.9.37
$ sudo make
$ sudo make install
```

読み込み設定
```
$ sudo mkdir -p /etc/depmod.d
$ echo 'search extra updates built-in' | sudo tee /etc/depmod.d/extra.conf
$ sudo depmod -a
$ sudo modprobe ddbridge
$ sudo reboot
```

確認
```
$ lspci | grep Multi
04:00.0 Multimedia controller: Digital Devices GmbH Device 000a
$ lsmod | grep ddb
ddbridge              155648  0
cxd2099                20480  1 ddbridge
dvb_core              151552  1 ddbridge
$ ls -l /dev/dvb
合計 0
drwxr-xr-x. 2 root root 120  3月 28 10:40 adapter0
drwxr-xr-x. 2 root root 120  3月 28 10:40 adapter1
drwxr-xr-x. 2 root root 120  3月 28 10:40 adapter2
drwxr-xr-x. 2 root root 120  3月 28 10:40 adapter3
```

### カードリーダー設定

```
$ sudo dnf install pcsc-tools
$ sudo systemctl enable pcscd
$ sudo systemctl restart pcscd
```

確認
```
$ sudo systemctl status pcscd
$ sudo pcsc_scan
Using reader plug'n play mechanism
Scanning present readers...
0: ACS ACR39U ICC Reader 00 00

Sat Mar 28 10:57:53 2020
 Reader 0: ACS ACR39U ICC Reader 00 00
  Card state: Card inserted,
~~~~~~~~
Japanese Chijou Digital B-CAS Card (pay TV)
```

### 録画コマンド導入

OKeyリポジトリの追加
```
$ sudo rpm -ivh http://repo.okay.com.mx/centos/8/x86_64/release/okay-release-1-3.el8.noarch.rpm
```

今回入れるdvbv5-zapはv4l-utilsに含まれている。
ChinachuのGitHubにある設定ファイルをとってくる。一チューナーで地上衛星両対応なのでファイルをまとめる。
```
$ sudo dnf install v4l-utils
$ cd ~/recwd
$ git clone https://github.com/Chinachu/dvbconf-for-isdb.git
$ cd dvbconf-for-isdb/conf
$ cat dvbv5_channels_isdbs.conf dvbv5_channels_isdbt.conf | sudo tee -a dvbv5_channels_isdb.conf
$ sudo vi /etc/yum.repos.d/CentOS-PowerTools.repo
// Enabledを1にする
$ sudo dnf install pcsc-lite-devel
```

確認
```
$ sudo dvbv5-zap -a 0 -c ./dvbv5_channels_isdb.conf -r -P 13
using demux 'dvb0.demux0'
reading channels from file './dvbv5_channels_isdb.conf'
tuning to 473142857 Hz
pass all PID's to TS
  dvb_set_pesfilter 8192
       (0x00) Signal= -34.03dBm C/N= 0.00dB UCB= 0
Lock   (0x1f) Signal= -41.72dBm C/N= 26.07dB UCB= 0
Lock   (0x1f) Signal= -41.34dBm C/N= 25.97dB UCB= 0
DVR interface '/dev/dvb/adapter0/dvr0' can now be opened
Lock   (0x1f) Signal= -41.34dBm C/N= 28.07dB UCB= 0 preBER= 0
```
いけた。次はMirakurun入れる。
