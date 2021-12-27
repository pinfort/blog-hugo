---
title: OctoberCMS を導入してみる
description: OctoberCMS導入記
date: 2019-04-25
slug: octobercms-installation
tags:
  - "OctoberCMS"
  - PHP
---
### LaravelベースのCMSであるOctoberCMSを使ってみたい。

```bash
composer create-project october/october blog
php artisan key:generate
```

とりあえず。PHPのcurl-extが無効だと怒られたので有効にしてね。

```bash
php artisan october:env
```

ほんとはoctober:installしてからのほうがいいらしいんだけど、いろいろあれなのでやってしまいます。これをするとoctober:installが使えませんが、個別でやればいいのでまあ。

```bash
cp .env .env.example
```

Keyだけ消しておきましょう

### Gitignore書いていきます

/blog/plugins/.gitignore
```
/*
!october
!.gitignore
```

/blog/modules/.gitignore
```
/*
!backend
!cms
!system
!.gitignore

```

/blog/themes/.gitignore
```
/*
!demo
!.gitignore

```

自前で書いたのはこれくらい。あとはデフォルトのGitignoreでいいんじゃないかな？必要が生じたら適宜書くけどね。また、デフォルトのpluginやmodule、themeはignoreしないようにしているけど、したほうがいいと書いてる記事もあるね。

### 設定を変えます
公式ドキュメントにあるように、disableCoreUpdatesをtrueにします。

/blog/config/cms.php
```
'disableCoreUpdates' => true, // changed
```

/blog/config/mail.php

書き換え前
```
'from' => ['address' => 'noreply@domain.tld', 'name' => 'OctoberCMS'],
```

書き換え後
```
'from' => ['address' => 'no-reply@blog.pinfort.me', 'name' => 'Pinfort blog'],
```

これを変えておかないと、（送信元ポリシーみたいなやつで）メールサーバーに拒否られたので。

### サーバー側にデプロイします

```bash
git clone hogehoge
curl -sS https://getcomposer.org/installer | php
php composer.phar install
cp .env.example .env
php artisan key:generate
```

.envにしてるとうまくkeyが設定されないので表示されるキーをviで.envに書く。

.envを編集する

マイグレーション
```
php artisan october:up
```

公開ディレクトリにindex.php等を公開
```
php artisan october:mirror /path/to/public_html/directory
```

デフォルトのやり方では全部のコードを公開ディレクトリ以下に置くようですが怖いので。とりあえずこれでデモページが見えるはずです。domain/backendでログイン画面が出ます。admin:adminでログインできるので、新しくユーザーを作って初期のadminユーザーは削除しておきましょう。
