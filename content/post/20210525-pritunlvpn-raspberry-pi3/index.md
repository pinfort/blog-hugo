---
title: pritunlでVPN構築 on Raspberry pi3
description: RasppberryPi 3上でPritunlを構築しました。
date: 2021-05-25
slug: pritunlvpn-raspberry-pi3
tags:
  - Linux
  - network
  - raspberrypi
---
## VPN 構築
今回は、ラズパイでVPN構築をしようと思います。今回構築しようとしているネットワークの大まかな図はこちらです。

    [Internet]
        |
    192.168.100.1/24
        |
    [   VPN server  ]
    |  192.168.1.2  |
    |  192.168.2.2  |
    -----------------
        |
    192.168.2.1/24 [My LAN]

## 材料
### ハードウェア
- 落ちていたRaspberry pi 3と、その電源ケーブル
- 落ちていたLANケーブル * 2
- USBネットワークアダプタ BUFFALO LUA4-U3-AGTE

### ソフトウェア
- Ubuntu 20.04 LTS
- pritunl

## 手順
### 初期設定
OSの初期設定をします。主にSSHとip固定の設定です。ラズパイについているＬＡＮポートを192.168.1.0/24のルーターに接続してipを固定します。

### ネットワークアダプタの接続
つなぐだけです。192.168.2.1/24側のルーターに接続し、ipを固定します。

### pritunlのインストール
pritunlにはarm64のバイナリがありません。ですので、ソースからインストールします。いくつか問題がありますので、修正しながらやっていきます。[README](https://github.com/pritunl/pritunl)を見ながらやります。

#### mongodbのインストール
READMEを見ると、最初にmongoDBをインストールしているようです。
[mongoDB公式](https://docs.mongodb.com/manual/tutorial/install-mongodb-on-ubuntu/)を見ながらやります。

```
wget -qO - https://www.mongodb.org/static/pgp/server-4.4.asc | sudo apt-key add -
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/4.4 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-4.4.list
sudo apt-get update

sudo apt-get install -y mongodb-org
# マニュアルではsudo dpkg --set-selectionsを使っていますが、apt-markを使います
sudo apt-mark hold mongodb-org
sudo apt-mark hold mongodb-org-server
sudo apt-mark hold mongodb-org-shell
sudo apt-mark hold mongodb-org-mongos
sudo apt-mark hold mongodb-org-tools

sudo systemctl start mongod
sudo systemctl status mongod
# 動いているので有効化
sudo systemctl enable mongod
```

#### golangのインストール
pritunlをビルドするためにgolangが必要です。最新は1.16.4でした。[Go公式](https://golang.org/doc/install#download)を見ながらやります。

```
wget https://golang.org/dl/go1.16.4.linux-arm64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.16.4.linux-arm64.tar.gz

# goがあることを確認
ls /usr/local/go
export PATH=$PATH:/usr/local/go/bin

# pathを追加
tee -a ~/.bashrc << EOF
export GOPATH=\$HOME/go
export PATH=/usr/local/go/bin:\$PATH
EOF
source ~/.bashrc

go version
# go version go1.16.4 linux/arm64
```

#### pythonのvertualenvの設定
Ubuntuにはすでにpython3が含まれていますので、virtualenvだけ入れます

```
sudo apt install python3-virtualenv -y
sudo mkdir -p /usr/lib/pritunl
sudo mkdir -p /var/lib/pritunl
sudo virtualenv /usr/lib/pritunl
```

#### goソースのビルド
ビルドに必要なソースをダウンロードしてビルド、配置します

```
go install github.com/pritunl/pritunl-dns@latest
go install github.com/pritunl/pritunl-web@latest
sudo cp -f ~/go/bin/pritunl-dns /usr/bin/pritunl-dns
sudo cp -f ~/go/bin/pritunl-web /usr/bin/pritunl-web
```

#### pythonソースビルド
最新バージョンは1.29.2664.67ですが、これはpython3に対応しておらず、python2用だったので、masterでやってみます。

```
export VERSION="master"
wget https://github.com/pritunl/pritunl/archive/$VERSION.tar.gz
tar xf $VERSION.tar.gz
rm $VERSION.tar.gz
cd ./pritunl-$VERSION
/usr/lib/pritunl/bin/python setup.py build
```

依存関係を追加してビルドして行きます。pynaclのインストールにはかなり時間がかかります。そのため、別にインストールしたほうがいいかもしれません。

```
sudo /usr/lib/pritunl/bin/pip3 install --upgrade pip
sudo apt-get install python3-dev libxml2-dev libxslt-dev -y
sudo apt-get install openvpn openssl net-tools psmisc ca-certificates -y

sudo /usr/lib/pritunl/bin/pip3 install pynacl
sudo /usr/lib/pritunl/bin/pip3 install -U -r requirements.txt
sudo /usr/lib/pritunl/bin/python setup.py install
sudo ln -sf /usr/lib/pritunl/bin/pritunl /usr/bin/pritunl
```

#### 起動

```
sudo systemctl daemon-reload
sudo systemctl start pritunl
sudo systemctl enable pritunl
sudo systemctl status pritunl
```

#### limit増加
[https://docs.pritunl.com/docs/configuration-5](https://docs.pritunl.com/docs/configuration-5)

```
sudo sh -c 'echo "* hard nofile 64000" >> /etc/security/limits.conf'
sudo sh -c 'echo "* soft nofile 64000" >> /etc/security/limits.conf'
sudo sh -c 'echo "root hard nofile 64000" >> /etc/security/limits.conf'
sudo sh -c 'echo "root soft nofile 64000" >> /etc/security/limits.conf'
```

#### setup

```
sudo pritunl setup-key
```

サーバーのipをブラウザで開く。問題なければ初期設定が見れる。（https設定をしていないので証明書エラーは出る。）keyを入力してsaveしたら

```
sudo pritunl default-password
```

表示された情報でログインする。ユーザー情報を編集して、public ipなど設定すればpritunlの初期設定は終了です。

### ネットワーク設定

とりあえず、WAN側のルーターで、以下の設定をします。
- 必要なファイアーウォールの穴あけ
  - udp [pritunlで作ったーサーバーのポート]
  - tcp 80
  - tcp 443
- 必要なNAPT設定
  - 80, 443, pritunlサーバーのポートを192.168.1.2へ転送

WAN側の設定ができたら、もうサーバーが外部から見えていると思うので、ドメインの設定をしたうえでpritunlの設定からドメインを入力するとよいでしょう。そうすれば、letsencryptで勝手に証明書を取ってくれます。

次に、netplanをいじります。
[参考](https://qiita.com/devzooiiooz/items/4e2e1c5264d8960eb412)
LAN側のインターフェースのpetplanの設定に以下を追加
```
routes:
- to: 192.168.2.0/24
  via: 192.168.2.1
```

```
sudo netplan apply
ip route
```

追加された。

pritunlでサーバーに192.168.2.0/24ルートを追加します。

すると、VPN接続端末から192.168.2.0/24にアクセスできました。
今回はここまでです。ありがとうございました。
