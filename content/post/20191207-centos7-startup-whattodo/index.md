---
title: サーバー建てて最初にやることめも（CentOS7）
description: サーバーの初期設定をしたので、最初にやることのメモを残しておきます。
date: 2019-12-07
slug: centos7-startup-whattodo
tags:
  - Linux
  - CentOS
---
久しぶりにサーバーの初期設定をやったので、覚書を残しておく。

## 環境
- さくらVPS
- CentOS7

## やったこと

1. sshの設定
  - <script src="https://gist.github.com/pinfort/2464fd2d7d5729ba2b95073c2e75c77e.js"></script>
  - ユーザー追加
  - 公開鍵認証設定
  - password authenticationの拒否
  - Rootログイン拒否
  - sshポート変更
  - sshd リロード

1. 一般ユーザーでログインしなおし

1. いろいろアップデート
```
sudo yum upgrade
```

1. SELinuxの設定

[SELINUX の有効化および無効化](https://access.redhat.com/documentation/ja-jp/red_hat_enterprise_linux/6/html/security-enhanced_linux/sect-security-enhanced_linux-working_with_selinux-enabling_and_disabling_selinux)

```
sudo yum install policycoreutils-python
sudo yum install setroubleshoot
```

上記の記事をそのままやると、ログインできなくなって詰んだのでpermissiveでいろいろ試験

- SELinux Disabled -> Permissive
- リブート
- sudo ausearch -m avc

```
type=AVC denied  { dyntransition } for comm="sshd" scontext=system_u:system_r:kernel_t:s0 tcontext=unconfined_u:unconfined_r:unconfined_t:s0 tclass=process permissive=1
```
一部省略

いろいろコンテキストがおかしそうなので
```
sudo restorecon -R /
sudo reboot
```

ポートを変更したのでsemanageで設定を変更しないといけないといけないことに気づいた

```
sudo semanage port --add --type ssh_port_t --proto tcp NEW_PORT
sudo semanage port --delete --type ssh_port_t --proto tcp 22
// ValueError: Port tcp/22 is defined in policy, cannot be deleted
```
消せないのかぁまあいいか

```
sudo setenforce 1
```

ログインできたのでさくらのパケットフィルタで22番ポートを閉じる
再起動

再起動してもログインできた！
監査ログに異常がないことを確認したのでSELinuxをEnforcingにし、もう一度再起動

正常にログインできました。やったぜ。

```
$ cat /etc/redhat-release
CentOS Linux release 7.7.1908 (Core)
```
