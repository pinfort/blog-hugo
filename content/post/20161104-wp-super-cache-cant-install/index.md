---
title: 【対処法】wp super cacheのインストールができない
description: wp super cacheのインストールにつまずいたので備忘録。
date: 2016-11-04
slug: "wp-super-cache-cant-install"
tags:
  - "Web Service"
---

wp super cacheのインストールにつまずいたので備忘録。

## はじめに
下の追記を見てください

advenced-cache.phpが更新できないとか出る。

ググって出てきたとおりにパーミッションを変更してもできない。（wp-config.php -> 644, wp-content -> 755）とか。

表示されているとおりにadvanced-cache.phpを書き換えてもできない。

wp-config.php で define(‘WP_CACHE’, true); を設定すると管理画面がメニューしか表示されなくなる。

 

## 解決法
path/to/wordpress/wp-content/plugins/wp-super-cache/advanced-cache.php を path/to/wordpress/wp-content/advanced-cache.php へコピー。

wp-config.php に define(‘WP_CACHE’, true); と define(‘WPCACHEHOME’, ‘path/to/wordpress/wp-content/plugins/wp-super-cache/’); を書き込む。（”編集が必要なのはここまでです”より上に書く。）

 

私の場合はこれでできました。パーミッションを変更してもできなかったのが謎。

パーミッション変更した場合はちゃんともとに戻しておく。

 

## 追記
原因がわかったので追記。

犯人はやっぱりSELinuxさんでした。

httpdの公開ディレクトリに指定したディレクトリには、（おそらく）自動で `httpd_sys_content_t` というコンテキストが設定されるようです。しかし、これを `httpd_sys_rw_content_t` に変えない限りApacheからの書き込みができないようです。

ということで、
```
semanage fcontext –a –t httpd_sys_rw_content_t “/path/to/wordpress(/.*)?”
```

を使ってコンテキストの設定を変更。
```
restorescon /path/to/wordpress -R
```
で設定の反映をしてからApache再起動でフィニッシュ。

現在設定されているコンテキストは ls -Z コマンドで確認できます。

いやあ、SELinuxさんは気難しいですね…
