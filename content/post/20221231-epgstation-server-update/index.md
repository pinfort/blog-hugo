---
title: 録画サーバーOS更新
description: CentOS7を使っていた録画サーバーのOSをUbuntuに変更して、再インストールします
date: 2022-12-31
slug: epgstation-server-update
---
## バックアップ
旧サーバーから、epgstationの機能を使ってDBのデータをバックアップします。録画データも別途退避します。サムネイル等はあきらめます。

```
npm run buckup HOGEHOGE
```

## OSインストール
CentOSの入っていたサーバーにUbuntuを入れます。インストールは通常通り行います。

## HDDマウント
録画データ保存用に別途HDDを用意しているのでマウントします。/etc/fstabを編集して自動でマウントされるようにします。

## ドライバ導入
[以前と同じ](https://blog.pinfort.me/posts/centos76-px-w3pe4-2/) 非公式ドライバを導入します。

make, gccが入っていなかったので導入します。

```
sudo apt install gcc make
```

必要なものをダウンロードします。

```
git clone https://github.com/nns779/px4_drv.git
wget http://plex-net.co.jp/download/pxw3pe4v1.4.zip
```

ファームウェア抽出ツール

```
cd px4_drv/fwtool/
make

gcc -O2 -Wall   -c -o fwtool.o fwtool.c
gcc -O2 -Wall   -c -o tsv.o tsv.c
gcc -O2 -Wall   -c -o crc32.o crc32.c
gcc -o fwtool fwtool.o tsv.o crc32.o
```

抽出実行

```
cd ../../
sudo apt install unzip
unzip pxw3pe4v1.4.zip
cd px4_drv/fwtool/
sudo ./fwtool ../../pxw3pe4v1.4/x64/PXW3PE4.sys it930x-firmware.bin
sudo cp -p it930x-firmware.bin /lib/firmware/
ls /lib/firmware/ | grep it930

it930x-firmware.bin
```



```
sudo apt install dkms
cd ../
vi dkms.install
```

dkms.installの内容
```
. ./dkms.conf
cp -a `pwd` /usr/src/$PACKAGE_NAME-$PACKAGE_VERSION
dkms add -m $PACKAGE_NAME -v $PACKAGE_VERSION
dkms build -m $PACKAGE_NAME -v $PACKAGE_VERSION
dkms install -m $PACKAGE_NAME -v $PACKAGE_VERSION
```

実行

```
sudo bash dkms.install
```

確認。認識できていそう。

```
sudo modprobe px4_drv
ls /dev | grep px4

px4video0
px4video1
px4video2
px4video3
```

再起動後も読み込まれるように設定

```
sudo vi /etc/modules-load.d/px4_drv.conf
```

confの内容
```
px4_drv
```

```
sudo reboot
```

再起動後に再確認。大丈夫そう

```
ls /dev | grep px4

px4video0
px4video1
px4video2
px4video3
```

## カードリーダー設定
ドライバの設定ができたので、カードリーダーの設定をします。

```
sudo apt install pcscd pcsc-tools libpcsclite-dev
sudo systemctl enable pcscd
pcsc_scan

Japanese Chijou Digital B-CAS Card (pay TV)
```

無事表示されたので大丈夫。

録画コマンドを入れる。

## 録画コマンド導入

```
sudo apt install pkg-config

mkdir arib25
cd arib25/
wget http://hg.honeyplanet.jp/pt1/archive/c44e16dbb0e2.zip
unzip c44e16dbb0e2.zip
cd pt1-c44e16dbb0e2/
cd arib25/
make clean
make
sudo make install


sudo apt install automake

cd ../../../
mkdir recpt1
cd recpt1
wget http://plex-net.co.jp/download/linux/Linux%20Driver_21.07.05.zip
unzip Linux\ Driver_21.07.05.zip
cd MyRecpt1/recpt1/
chmod +x ./autogen.sh
./autogen.sh
chmod +x ./configure
./configure --enable-b25
make
sudo make install
```

録画チェックをしてみる。なんか良さそうなのでOK
```
recpt1 -b --device /dev/px4video2 22 10 test1.ts
```

## docker導入

OS標準で入るdockerが古すぎるらしいので消して入れなおします。
```
sudo snap remove docker
sudo reboot
```

[https://japan.zdnet.com/article/35190536/](https://japan.zdnet.com/article/35190536/)
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get install apt-transport-https ca-certificates curl gnupg lsb-release -y
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io -y
```

epgstation用のユーザーをつくり、dockerを触れるようにする
```
adduser epgstation
sudo usermod -aG docker epgstation
```

## epgstation構築
[https://github.com/l3tnun/docker-mirakurun-epgstation](https://github.com/l3tnun/docker-mirakurun-epgstation)
dockerで構築する。

ホストのpcscdは停止しなければならないようなので停止。docker-composeをsudoで実行しないといけないことに気づいたのでグループに追加
```
sudo passwd epgstation
sudo usermod -aG sudo epgstation

curl -sf https://raw.githubusercontent.com/l3tnun/docker-mirakurun-epgstation/v2/setup.sh | sh -s
cd docker-mirakurun-epgstation
```

vi tuners.yml
今回、BSは使用しないのでdisabledにしておきます
```
- name: PX4-S1
  types:
    - BS
    - CS
  command: recpt1 --device /dev/px4video0 --lnb 15 <channel> - -
  decoder: arib-b25-stream-test
  isDisabled: true

- name: PX4-S2
  types:
  command: recpt1 --device /dev/px4video1 --lnb 15 <channel> - -
  decoder: arib-b25-stream-test
  isDisabled: true

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

このままだとrecpt1がなくて録画できないので、以下のサイトに従ってdockerFileを書き換えます
[https://qiita.com/nanbuwks/items/640ee4405e1fdd2ca497](https://qiita.com/nanbuwks/items/640ee4405e1fdd2ca497)

```
docker-compose build
```

ビルドには時間がかかるのでただ待ちます。できたら起動します。
```
sudo docker-compose up -d
```

地上波のチャンネルスキャンを行うので、channels.ymlの地上波部分を削除してAPIをたたきます。
```
curl -X PUT "http://localhost:40772/api/config/channels/scan"
```

docker cpをして、DBのバックアップファイルをコンテナにコピーし、リストアしてから削除します。
```
npm run restore HOGEHOGE
```

epgstationのconfig.ymlはいい感じに編集します。
もろもろ終えれば再起動します。
```
sudo docker-compose down
sudo docker-compose up -d
```

これでだいたいの移行作業は終了だと思います。
