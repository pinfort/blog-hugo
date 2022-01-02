---
title: XSERVERでRuby on Railsを動かす　その2
description: XSERVERでRuby on Railsを動かしてみた記録第二話です。すべての工程を最終話にまとめていますので、この記事を見る必要はありません。
date: 2018-09-02
slug: run_ruby_on_rails_on_xserver_2
tags:
  - "Ruby"
  - "Web service"
  - "XSERVER"
series:
  - "run ruby on rails on xserver"
---
> この記事は試行錯誤の記録です！railsを動かすには、[その4（結論編）](https://blog.pinfort.me/posts/run_ruby_on_rails_on_xserver_4)だけを見てもらえれば解決します！

前回の続き。

~~readlineのほかにedit textというのもいりそうだとわかった。~~

~~## libeditのインストール~~

~~[https://thrysoee.dk/editline/](https://thrysoee.dk/editline/)~~

~~wget https://thrysoee.dk/editline/libedit-20180525-3.1.tar.gz~~

~~ncurses.hがないとかいろいろ言われる~~

~~ncursesのコンパイルoptionが足りてない？らしいので再コンパイル~~

~~./configure --prefix=/home/ユーザー名/opt --with-shared~~

~~なんかコンパイルに失敗するのでパッチを当てる~~

~~[https://github.com/tydk27/dotfiles/blob/master/patch/ncurses-6.0-gcc5.patch](https://github.com/tydk27/dotfiles/blob/master/patch/ncurses-6.0-gcc5.patch)~~

~~だめやん~~

**ローカルで同じ環境を再現し、同じディレクトリ構造でRubyをコンパイルしてから持っていったら何とかなりました**

gemを入れるときは毎回

```
gem install xxxx -- --with-ldflags='-L/home/ユーザー名/opt/lib'
```

のようにldflagが必要です

mysql-develがひつようらしい。

## mysql-develのインストール

[https://downloads.mariadb.org/mariadb/5.5.56/](https://downloads.mariadb.org/mariadb/5.5.56/)

ここでコンパイル済みが入手できる。もうmakeしなくていい。

## bundle install
グローバルのbundleでは失敗するので

```
gem install bundler -- --with-ldflags='-L/home/ユーザー名/opt/lib'
```

で、bundle install

ldflagsが必要な奴は個別でやる。

## ./bin/rails s
うごいた！

つぎ！[run_ruby_on_rails_on_xserver_3](run_ruby_on_rails_on_xserver_3)
