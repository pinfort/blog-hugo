---
title: CentOS8.1でMirakurun
description: CentOS8.1でMirakurun入れるまで
date: 2020-03-28
slug: centos81-mirakurun
tags:
  - CentOS
  - Linux
  - PC-tv
series:
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

前回までで録画コマンドは導入済みです。

```
$ sudo mkdir /usr/local/dvbconf-for-isdb
$ sudo mkdir /usr/local/dvbconf-for-isdb/conf
$ sudo cp ~/recwd/dvbconf-for-isdb/conf/dvbv5_channels_isdb.conf /usr/local/dvbconf-for-isdb/conf/
```

```
$ curl -sL https://rpm.nodesource.com/setup_12.x | sudo bash -
$ sudo yum install nodejs
$ sudo npm update -g
$ sudo npm install pm2 -g
$ sudo npm install mirakurun -g --production
$ sudo npm install arib-b25-stream-test -g --unsafe
$ sudo mirakurun init
[PM2] App [mirakurun-server] launched (1 instances)
```

設定書き換え
```
$ sudo mirakurun config tuners
```

```
- name: MaxM4
  types:
    - GR
    - BS
    - CS
  command: dvbv5-zap -a 0 -c /usr/local/dvbconf-for-isdb/conf/dvbv5_channels_isdb.conf -r -P <channel>
  dvbDevicePath: /dev/dvb/adapter0/dvr0
  decoder: arib-b25-stream-test
  isDisabled: false
 
- name: MaxM4
  types:
    - GR
    - BS
    - CS
  command: dvbv5-zap -a 1 -c /usr/local/dvbconf-for-isdb/conf/dvbv5_channels_isdb.conf -r -P <channel>
  dvbDevicePath: /dev/dvb/adapter1/dvr0
  decoder: arib-b25-stream-test
  isDisabled: false
 
- name: MaxM4
  types:
    - GR
    - BS
    - CS
  command: dvbv5-zap -a 2 -c /usr/local/dvbconf-for-isdb/conf/dvbv5_channels_isdb.conf -r -P <channel>
  dvbDevicePath: /dev/dvb/adapter2/dvr0
  decoder: arib-b25-stream-test
  isDisabled: false
 
- name: MaxM4
  types:
    - GR
    - BS
    - CS
  command: dvbv5-zap -a 3 -c /usr/local/dvbconf-for-isdb/conf/dvbv5_channels_isdb.conf -r -P <channel>
  dvbDevicePath: /dev/dvb/adapter3/dvr0
  decoder: arib-b25-stream-test
  isDisabled: false
```

```
$ sudo mirakurun restart
```

チャンネルスキャン。
```
$ curl -X PUT "http://localhost:40772/api/config/channels/scan"
channel: "13" ...
-> 2 services found.
-> {"name":"ＮＨＫ総合","type":"GR","channel":"13"}
```
行けてそうです。

BS, CSの自動設定スクリプトを入れます。

```
$ cd ~/recwd
$ git clone https://github.com/stz2012/epgdump
$ cd epgdump
$ make
$ sudo make install
$ cd ../
$ git clone https://gist.github.com/479b7da80063e39d1ca2cf467e40e290.git
$ cd 479b7da80063e39d1ca2cf467e40e290
$ sudo vi scan_ch_mirak.sh
```

一部設定をdvbv5-zap用にコメントを切り替える。confファイルのパスも書き換える。
いったんMirakurunを停止してチューナーを開放する。
うまく実行できなかったのでvisudoでsecure_pathに/usr/local/binを加えた。
※注意　BS, CSのスキャンを行ったのだが、チャンネル名称がBS1\_1のようになっていて、dvbv5\_channels\_isdb.confはBS01\_1のようになっているので、そのままでは動かなかった。BS1\_1のようになっているchannels.ymlをBS01\_1のように編集して対処。

BSの一部チャンネルが映らなかった。おそらく自宅が特殊な環境だったのだが、dvbv5\_channels\_isdb.confとchannels.ymlに手動で設定を加えて対処。

```
$ sudo mirakurun stop
$ sudo bash scan_ch_mirak.sh
$ sudo cp channels.yml /usr/local/etc/mirakurun/channels.yml
$ sudo mirakurun restart
```

```
$ sudo pm2 install pm2-logrotate

$ cat << 'EOT' | sudo tee /etc/logrotate.d/mirakurun
/usr/local/var/log/mirakurun.stdout.log
/usr/local/var/log/mirakurun.stderr.log
/{
daily
compress
rotate 7
missingok
notifempty
}
EOT

sudo mirakurun restart
```

ちゃんと起動してればいい。
