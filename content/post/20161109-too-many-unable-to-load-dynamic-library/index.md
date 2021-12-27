---
title: PHP で Unable to load dynamic libraryがいっぱい出る
description: PHPをアプデしたら大量にWarningが出たので解決法備忘録
date: 2016-11-09
slug: too-many-unable-to-load-dynamic-library
tags:
  - "PHP"
---
PHPをアプデしたら大量にWarningが出たので解決法備忘録。

環境
CentOS 7

PHP 5.4 → 7.0 へのアプデ

状況
以下のようなWarningがいっぱい（おそらく追加モジュール全部）出る

```
PHP Warning:  PHP Startup: Unable to load dynamic library ‘usr/lib64/php/modules/sysvsem.so’ – usr/lib64/php/modules/sysvsem.so: cannot open shared object file: No such file or directory in Unknown on line 0
```

解決法
エラーメッセージを見ればわかるように usr/lib64/php/modules/ ディレクトリを参照している。

つまり、*相対パス*になってる！！

/etc/php.ini の extension_dir を usr/lib64/php/modules/ から /usr/lib64/php/modules/ に修正した。

これでエラーが出なくなった。
