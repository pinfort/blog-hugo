---
title: mastodonのDBをDockerからはがし、別のサーバーに分ける
description: Dockerで運用されているmastodonインスタンスからDBだけ分離し、ほかのサーバーに移行する方法。
slug: mastodon-detach-database-from-docker-to-other-server
date: 2019-02-01
tags:
  - "Web service"
  - mastodon
---
- DBサーバーにするサーバーにpostgresqlをインストール
- 初期設定等を済ませる
- postgresqlの外部からのアクセスを許可
- listen_addresses='localhost,サーバーのip'
- firewalldで不特定多数からのhttpやhttpsのアクセスを拒否（httpやhttpsをpublic zoneから削除）しつつ、Webサーバーからのアクセスを許可（webサーバーのipをtrustゾーンに追加）
- firewalldをpermanentに保存しreload
- postgresqlも再起動
- webサーバーにpsql（クライアント）をインストールし、DBサーバーにアクセスできることを確認。psql -h DBサーバーのip -U postgres
- できたら、webサーバーのnginxを一旦停止
- docker exec mastodon_db_1 pg_dumpall -U postgres > 任意の名前.sql 
- 念のためdocker-compose downでwebサーバー側のmastodonを停止
- sqlファイルをDBサーバーにコピー
- DBサーバーのDBにインポート
- psql -U postgres -f 任意の名前.sql
- postgresユーザーにパスワードをかけるとかする
- webサーバー側の.env.productionのDBホスト、パスワード等いい感じに書き換える
- Webサーバーをdocker-compose buildする
- docker-compose up -dする
- データベースのユーザーは、dockerの中ではpostgresになっていると思うが、移行するときにmastodonユーザーなどほかのユーザーで運用したいと思うときは、いったん上記のようにpostgresユーザーでpostgresデータベースにインポートし、データベースをリネームなどしたほうが混乱がなくてよい。
