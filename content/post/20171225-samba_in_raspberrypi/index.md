---
title: ラズパイでsamba
description: ラズパイでsamba建てたい。めも。
date: 2017-12-25
slug: samba_in_raspberrypi
tags:
  - "raspberrypi"
---
## ファイルサーバーにしたいのでまず外付けHDDを接続してマウントする

参考
- [https://qiita.com/NoTASK/items/8521d237bcacf75a37ba](https://qiita.com/NoTASK/items/8521d237bcacf75a37ba)
- [https://qiita.com/wbh/items/90288f44b62a36d24f0a](https://qiita.com/wbh/items/90288f44b62a36d24f0a)
-[https://engetu21.hatenablog.com/entry/2015/02/01/205818](https://engetu21.hatenablog.com/entry/2015/02/01/205818)


```
sudo apt-get install cifs-utils
sudo mkdir /mnt/hdd1
sudo chmod 777 /mnt/hdd1
```

ntfs扱う用のパッケージインスコ

```
sudo apt-get install ntfs-3g
sudo fdisk -l
```

マウント対象確認。今回は/dev/sda1だった。

マウント

```
sudo mount /dev/sda1 /mnt/hdd1
```

自動マウント設定したい

UUID確認

```
sudo blkid /dev/sda1
```

自動マウント設定

```
sudo vi /etc/fstab
UUID="XXXXXXXXXXXXX" /mnt/hdd1 ntfs-3g defaults 0 2
```

再起動してマウントされていればOK

## SAMBA入れる


```
sudo apt-get install samba
```

共有用セクション書く

```
sudo nano /etc/samba/smb.conf
```

[http://www.atmarkit.co.jp/ait/articles/0901/28/news153.html](http://www.atmarkit.co.jp/ait/articles/0901/28/news153.html)
```
[nas]
path=/mnt/hdd1
read only=no
comment = "main NAS server"
```

samba用パスワード設定

```
sudo pdbedit -a pi
```

これでいけた。

winからネットワークドライブの割当を選択したときに、「別の資格情報を利用して接続する」を選択するのを忘れずに
