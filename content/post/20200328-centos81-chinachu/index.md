---
title: CentOS8.1でChinachu
description: CentOS8.1でChinachu
date: 2020-03-28
slug: centos81-chinachu
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

```
$ sudo adduser chinachu
$ su -
# su - chinachu
$ git clone https://github.com/Chinachu/Chinachu.git ~/chinachu
$ cd ~/chinachu/
$ git branch
* gamma

$ ./chinachu installer
Chinachu Installer Menu:
[!] These are installed under all /home/chinachu/chinachu/...
[!] Recommend the Auto installation.
1) Auto (full)          3) Node.js Environment  5) ffmpeg
2) submodule            4) Node.js Modules
what do you install? > 1
selected: Auto (full)
```

```
$ cp config.sample.json config.json
$ vi config.json
```
configを書く。

```
$ echo [] > rules.json
$ sudo pm2 start processes.json
```

動かない。shasum入れてなかった。

```
$ cd ~/chinachu/chinachu
$ rm -rf .nave/
$ ./chinachu installer
Chinachu Installer Menu:
[!] These are installed under all /home/chinachu/chinachu/...
[!] Recommend the Auto installation.
1) Auto (full)          3) Node.js Environment  5) ffmpeg
2) submodule            4) Node.js Modules
what do you install? > 1
# pm2 delete processes.json
# pm2 start processes.json
# pm2 startup
```

いつまでたっても番組表の取得が終わらないと思ったら
```
2020-03-28T22:55:34.276+09:00 warn: db `/usr/local/var/db/mirakurun/programs.json` integrity check has failed
```
みたいなエラーが出ていて、これはSELinuxのせいだったのでルールを作成して回避。
