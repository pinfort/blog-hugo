---
title: CentOS 8.1 でMinidlna(ReadyMedia)
description: CentOS 8.1でMiniDLNAをつかって録画鯖の中身を鑑賞する
date: 2020-04-09
slug: centos-81-minidlna-readymedia
tags:
  - CentOS
  - Linux
  - PC-tv
series:
  - "CentOS8.1-DD-Max-M4"
---
CentOSでの録画サーバー構築の一環で、DLNAサーバーを入れようと思います。

## 環境

### PC

中古のDiginnos Raytrek LT K600

```
$ cat /etc/redhat-release
CentOS Linux release 8.1.1911 (Core)
```

## やったこと
EPELはすでに入っているのでNux Dextopを入れる。CentOS 8用がまだないが、7用で代用できるかなぁ？

```
$ sudo yum install http://li.nux.ro/download/nux/dextop/el7/x86_64/nux-dextop-release-0-5.el7.nux.noarch.rpm
```

minidlnaを入れます。
```
$ dnf search minidlna
Nux.Ro RPMs for general desktop use             1.1 MB/s | 4.2 MB     00:03
メタデータの期限切れの最終確認: 0:00:02 時間前の 2020年04月09日 00時38分55秒 に 実施しました。
=========================== 名前 完全一致: minidlna ============================
minidlna.x86_64 : Lightweight DLNA/UPnP-AV server targeted at embedded systems
========================== 名前 & 概要 一致: minidlna ==========================
minidlna-debuginfo.x86_64 : Debug information for package minidlna
```

ffmpegが必要なようなので先にそれを入れる。（Nux-dextopにもffmpegは入っているが、必要なものいろいろがないと怒られたのでRPM fusionを入れた）

RPM fusionリポジトリの追加
```
$ sudo dnf install https://download1.rpmfusion.org/free/el/rpmfusion-free-release-8.noarch.rpm
$ sudo dnf install https://download1.rpmfusion.org/nonfree/el/rpmfusion-nonfree-release-8.noarch.rpm
$ sudo dnf install ffmpeg
$ ffmpeg -version
ffmpeg version 4.2.2 Copyright (c) 2000-2019 the FFmpeg developers
built with gcc 8 (GCC)......
```

```
$ sudo dnf install minidlna
$ sudo  vi /etc/minidlna.conf //雑に編集
$ sudo systemctl start minidlna
$ sudo systemctl enable minidlna
$ sudo firewall-cmd --add-port=8200/tcp --zone=public --permanent
$ sudo firewall-cmd --add-port=1900/udp --zone=public --permanent
$ sudo firewall-cmd --reload
```
さくっと録画鯖の中身が見れるようになった。
