---
title: Connectiontest.devをリリースしました
description: 今日、新しいWebサービス、Connectiontest.devをリリースしました。簡単なものなので、Webサービスといっていいのかもわかりませんが...
slug: connectiontest-released
date: 2019-03-29
tags:
  - "Web service"
  - "PHP"
---
今日、新しいWebサービス、Connectiontest.devをリリースしました。簡単なものなので、Webサービスといっていいのかもわかりませんが...

Connectiontest.devは、[https://connectiontest.dev](https://connectiontest.dev) で展開されるサービスで、ただひたすらにリクエストされた情報をjsonに整形してレスポンスします。このドメイン配下のあらゆるURLにアクセス可能で、複数のHTTP動詞にも対応しています。

## 注意

デプロイしているレンタルサーバーの都合で、HTTPヘッダがいろいろ書き換えられています。サーバー移すか検討中ですが、レンサバが一番費用的に楽なんだよなぁ...

## なぜ作ったのか

テストを書いていた時に、外部のサーバーにリクエストを送る部分のテストをいい感じに書きたかったので、作りました。あと、何かサクっと作りたかったので。

## Httpbinは?

正直、Httpbinの存在を知って、POSTとか、GET以外もできるのでもうよくねと思ったりしたんですが、パスが決まっているのでめんどくさいかなと思いました。パスが自由だと、リクエストのホストを挿げ替えるだけで200が返ってくるので...いやまあ特に意味なくても作りたいじゃん？
