---
title: CentOS7.6 で chinachu（第二回）Chinachu導入編
description: CentOS7.6 で chinachuを動かしたい第二回です。
date: 2019-06-07
slug: centos76-chinachu-install-2
tags:
  - Linux
  - PC-tv
series:
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

[前回の記事](/posts/centos76-chinachu-install-1)

前回、Mirakurunを導入できましたので、次にChinachuを導入していきます。

### Chinachuユーザーの追加

```bash
$ sudo adduser chinachu
$ su -
# su - chinachu
```

### Chinachuのダウンロード

```bash
$ git clone https://github.com/Chinachu/Chinachu.git ~/chinachu
$ cd ~/chinachu/
$ git branch
* gamma
```

### インストール

```bash
$ ./chinachu installer
Chinachu Installer Menu:
[!] These are installed under all /home/chinachu/chinachu/...
[!] Recommend the Auto installation.
1) Auto (full)          3) Node.js Environment  5) ffmpeg
2) submodule            4) Node.js Modules
what do you install? > 1
...
ffmpeg --> done.
```

### 録画フォルダの作成

録画保存用HDDを/mnt/hdd1にマウントしているので、/mnt/hdd1/recordedディレクトリを作成しておきましょう。

```bash
$ mkdir /mnt/hdd1/recorded
```

### 設定

```bash
$ cp config.sample.json config.json
$ vi config.json
```

ユーザー変更
```json
"uid": null, -> "uid": "chinachu",
```

録画ディレクトリ変更

```json
"recordedDir" : "./recorded/", -> "recordedDir" : "/mnt/hdd1/recorded/",
```

録画ファイルの名前変更
```json
"recordedFormat": "[<date:yymmdd-HHMM>][<type><channel>][<channel-name>]<title>.m2ts", -> "recordedFormat": "<category>/<title>/[<date:yymmdd-HHMM>][<type><channel>][<channel-name>]<title>#<episode>.m2ts",
```

空き容量減少時の挙動変更
```json
"storageLowSpaceAction": "remove", -> "storageLowSpaceAction": "stop",
```

ツイッター通知用設定追加

```json
"operTweeter" : true,
"operTweeterAuth" : {
    "consumerKey"       : "",
    "consumerSecret"    : "",
    "accessToken"       : "",
    "accessTokenSecret" : ""
  },
  "operTweeterFormat": {
    "start": "●REC <channelname> <fullTitle> #pinfort_rec_notifier",
    "end": "●FINISH <channelname> <fullTitle> #pinfort_rec_notifier"
  }
```

録画予約用空ファイル作成

```bash
$ echo "[]" > rules.json
```

### ログローテーション設定

ログローテーション用のライブラリはMirakurunを入れるときに入れたので、設定だけします。

```bash
# vi /etc/logrotate.d/chinachu
```

ファイル内容
```bash
/usr/local/var/log/chinachu-operator.stderr.log
/usr/local/var/log/chinachu-operator.stdout.log
/usr/local/var/log/chinachu-wui.stderr.log
/usr/local/var/log/chinachu-wui.stdout.log
{
  weekly
  compress
  rotate 4
  missingok
  notifempty
}
```

### 起動確認

```bash
# su - chinachu
$ cd ~/chinachu
$ ./chinachu service wui execute
Client {
  basePath: '/api',
  priority: 0,
  host: '',
  port: 40772,
  socketPath: '/var/run/mirakurun.sock',
  userAgent: 'Chinachu/0.10.0-gamma.0 (wui)',
  _userAgent: 'MirakurunClient/2.11.0 Node/v10.16.0 (linux)' }
...
```
起動できましたね。

### EPG取得テスト

```bash
$ ./chinachu update
$ ./chinachu update
Client {
  basePath: '/api',
  priority: 0,
  host: '',
  port: 40772,
  socketPath: '/var/run/mirakurun.sock',
  userAgent: 'Chinachu/0.10.0-gamma.0 (scheduler)',
  _userAgent: 'MirakurunClient/2.11.0 Node/v10.16.0 (linux)' }
7 Jun 13:49:54 - GETTING EPG from Mirakurun.
7 Jun 13:49:54 - Mirakurun is OK.
7 Jun 13:49:54 - Mirakurun -> services: 39
7 Jun 13:49:54 - Mirakurun -> services: 39 (excluded)
7 Jun 13:49:54 - Mirakurun -> sorted services: 0
7 Jun 13:49:54 - Mirakurun -> programs: 8933
7 Jun 13:49:54 - Mirakurun -> tuners: 2
7 Jun 13:49:54 - WRITE: /home/chinachu/chinachu/data/schedule.json
7 Jun 13:49:54 - RUNNING SCHEDULER.
7 Jun 13:49:54 - TUNERS: {"GR":2}
7 Jun 13:49:54 - MATCHES: 0
7 Jun 13:49:54 - DUPLICATES: 0
7 Jun 13:49:54 - CONFLICTS: 0
7 Jun 13:49:54 - SKIPS: 0
7 Jun 13:49:54 - RESERVES: 0
7 Jun 13:49:54 - WRITE: /home/chinachu/chinachu/data/reserves.json
```

正常に動いてますね。

### PM2にchinachuを登録

```bash
$ su -
# cd /home/chinachu/chinachu
# pm2 start processes.json
# pm2 save
```

chinachu-operatorとchinachu-wuiができるのですが、なんとエラーになってしまいました。
shasumというコマンドがないために、CentOSではエラーになってしまうようです。

shasumコマンド追加
```bash
# yum install perl-Digest-SHA
```
.nave以下削除、chinachu再インストール
```bash
$ cd ~/chinachu/chinachu
$ rm -rf .nave/
$ ./chinachu installer
Chinachu Installer Menu:
[!] These are installed under all /home/chinachu/chinachu/...
[!] Recommend the Auto installation.
1) Auto (full)          3) Node.js Environment  5) ffmpeg
2) submodule            4) Node.js Modules
what do you install? > 1
...
```

PM2から削除、再度登録

```bash
# pm2 delete processes.json
# pm2 start processes.json
```

やっと全部onlineになりましたね。とりあえず導入完了かな？
