---
title: CentOS7.6 で chinachu（第一回）Mirakurun導入編
description: CentOS7.6 で chinachuを動かしたい第一回です。
date: 2019-06-06
slug: centos76-chinachu-install-1
tags:
  - Linux
  - PC-tv
categories:
  - "CentOS7.6-Chinachu"
---
### やりたいこと

CentOS7.6環境でChinachuを動かしたい。

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

[前シリーズ](/categories/centos76-px-w3pe4)までで、録画コマンドによる録画はできるようになりました。今回からは、MirakurunとChinachuを導入して、録画鯖として使えるようにするところまでやっていきます。

### NodeJSの導入

NodeJS 12が出てるので12を入れてみます。

```bash
$ curl -sL https://rpm.nodesource.com/setup_12.x | sudo bash -
...
## Run `sudo yum install -y nodejs` to install Node.js 12.x and npm.
## You may also need development tools to build native addons:
     sudo yum install gcc-c++ make
## To install the Yarn package manager, run:
     curl -sL https://dl.yarnpkg.com/rpm/yarn.repo | sudo tee /etc/yum.repos.d/yarn.repo
     sudo yum install yarn

$ sudo yum install nodejs
...
完了しました!
```

### PM2の導入

```bash
$ sudo npm install pm2 -g
...
added 319 packages from 257 contributors in 9.62s
```

### Mirakurunの導入

```bash
$ sudo npm install mirakurun -g --production
...
Version: node@v12.4.0 [NG] Expected: ^8.9.4 < 9 || ^10.11.0 < 11
```

あら。Node v12はサポート外とな。v10を入れましょう。

### NodeJS v10の導入

```bash
$ sudo yum remove nodejs
$ curl -sL https://rpm.nodesource.com/setup_10.x | sudo bash -
$ sudo yum clean all
$ sudo yum install nodejs
...
---> パッケージ nodejs.x86_64 2:10.16.0-1nodesource を インストール
...
# v10になっていることを確認します
...
完了しました!
```

### PM2の導入

いったんNodejs消したので一応やっておきます

```bash
$ sudo npm install pm2 -g
```

### Mirakurunの導入

再チャレンジ

```bash
$ sudo npm install mirakurun -g --production
...
added 170 packages from 101 contributors in 3.268s
```

入りましたね。

### Mirakurunのサービス登録と起動

```bash
$ sudo mirakurun init
...
[PM2] App [mirakurun-server] launched (1 instances)
|-------------------------------------------------------------|
| Name             | id | mode | status | ? | cpu | memory    |
|------------------+----+------+--------+---+-----+-----------|
| mirakurun-server | 0  | fork | online | 0 | 0%  | 15.9 MB   |
|-------------------------------------------------------------|
 Use `pm2 show <id|name>` to get more details about an app
[PM2] Saving current process list...
[PM2] Successfully saved in /root/.pm2/dump.pm2
```

Onlineなので起動できてます。

### Mirakurunの設定

特にポート等は変更しないので、とりあえずチューナー設定をします。

```bash
$ sudo mirakurun config tuners
```

ファイル内容

```bash
- name: PX4-S1
  types:
    - BS
    - CS
  command: recpt1 --device /dev/px4video0 --lnb 15 <channel> - -
  decoder: arib-b25-stream-test
  isDisabled: false

- name: PX4-S2
  types:
    - BS
    - CS
  command: recpt1 --device /dev/px4video1 --lnb 15 <channel> - -
  decoder: arib-b25-stream-test
  isDisabled: false

- name: PX4-T1
  types:
    - GR
  command: recpt1 --device /dev/px4video2 <channel> - -
  decoder: arib-b25-stream-test
  isDisabled: false

- name: PX4-T2
  types:
    - GR
  command: recpt1 --device /dev/px4video3 <channel> - -
  decoder: arib-b25-stream-test
  isDisabled: false
```

書けたら保存。

スクランブル解除用のコマンドを入れます。unsafeを付けないと入りませんでした。

```bash
$ sudo npm install arib-b25-stream-test -g --unsafe
```

チャンネルスキャンをしていきます。

```bash
$ curl -X PUT "http://localhost:40772/api/config/channels/scan"
...
-> no signal. [Error: no available tuners]
```

なんでだ？と思ったら、recpt1がrootで使えないと発生するようです。実際、sudoを付けると

```bash
$ sudo recpt1 -b --device /dev/px4video2 22 5 test1.ts
sudo: recpt1: コマンドが見つかりません
```

となりました。ということで、visudoで追記します。

```bash
$ sudo visudo
```

編集内容

secure_pathに:/usr/local/binを追記します。
注）たいてい/usr/local/binに入ると思いますが、一応which recpt1で場所を確認したほうがいいかもしれません。

Mirakurunを再起動した後に、チャンネルスキャン（リベンジ）をしていきます。

```bash
$ sudo mirakurun restart
$ curl -X PUT "http://localhost:40772/api/config/channels/scan"
...
-> no signal. [Error: no available tuners]
```

なんでだろう？
パスは通したと思ったんですけど無理そうなのでフルパス指定にしてみます。

```bash
- name: PX4-S1
  types:
    - BS
    - CS
  command: /usr/local/bin/recpt1 --device /dev/px4video0 --lnb 15 <channel> - -
  decoder: arib-b25-stream-test
  isDisabled: true

- name: PX4-S2
  types:
    - BS
    - CS
  command: /usr/local/bin/recpt1 --device /dev/px4video1 --lnb 15 <channel> - -
  decoder: arib-b25-stream-test
  isDisabled: true

- name: PX4-T1
  types:
    - GR
  command: /usr/local/bin/recpt1 --device /dev/px4video2 <channel> - -
  decoder: arib-b25-stream-test
  isDisabled: false

- name: PX4-T2
  types:
    - GR
  command: /usr/local/bin/recpt1 --device /dev/px4video3 <channel> - -
  decoder: arib-b25-stream-test
  isDisabled: false
```

あと、BSは使わないのでdisabledにしてます。（アンテナがないので）

いけっ！

```bash
$ curl -X PUT "http://localhost:40772/api/config/channels/scan"
channel scanning... (type: "GR")

channel: "13" ...
-> 3 services found.
-> {"name":"ＮＨＫＥテレ","type":"GR","channel":"13"}

channel: "14" ...
-> 3 services found.
-> {"name":"読売テレビ","type":"GR","channel":"14"}
...
```

きましたねぇ。フルパスにするのが一番確実でよいですね。

### log rotateの導入

```bash
$ sudo pm2 install pm2-logrotate
```

入れて設定しておきます。

```bash
$ sudo vi /etc/logrotate.d/mirakurun
```

ファイル内容

```bash
/usr/local/var/log/mirakurun.stdout.log
/usr/local/var/log/mirakurun.stderr.log
/{
  daily
  compress
  rotate 7
  missingok
  notifempty
}
```

これでMirakurunは終わりですかね。次回はいよいよChinachuを入れていきます。
