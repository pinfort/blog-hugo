---
title: SSH鍵認証ができない
description: 公開鍵認証ができない事象を解決したので備忘録。
date: 2016-11-07
slug: can-not-login-with-pubkey
tags:
  - "Linux"
---
公開鍵認証ができない事象を解決したので備忘録。

## 症状

特定のユーザーだけ公開鍵認証ができない。

```bash
/usr/bin/sshd -d
```
コマンドを使ってデバッグモードでsshdを起動するとログインできる。

```bash
/var/log/secure
```
には、特に有用な情報は見つけられない。

パーミッション設定はすべて正しい。

```bash
/home/username -> 755
/home/username/.ssh -> 700
/home/username/.ssh/authorized_keys -> 600
```

## 解決法

結論としては、ホームディレクトリ、.sshディレクトリ、authorized_keysに余計なコンテキストが設定されていたのが原因だった。

httpdによって公開するディレクトリをユーザーホーム以下の任意のディレクトリに設定したとき、httpd_sys_content_t　のコンテキストを誤って /home/username 以下全てに設定してしまっていた。

```bash
restorescon /home/username -R
```
を使ってリセットすることで解決した。

```bash
semanage fcontext -l | grep httpd
```
で
```bash
/home/username
```
に設定されているコンテキストを確認し、設定されていないか、正しく設定されているならば

```bash
restorecon /home/username -R
```
でOK。

```bash
/home/username
```
に　
```bash
httpd_sys_content_t,
httpd_sys_rw_content_t
```
等が設定されているようであれば
```bash
semanage fcontext –d "/home/username(/.*)?"
```
などを使って設定を削除してから
```bash
restorecon /home/username -R
```
を行う。

ファイル、フォルダのコンテキストは
```bash
ls -Z /home
```
で確認できる。
