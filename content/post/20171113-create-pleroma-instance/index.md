---
title: centos7 でpleromaインスタンスを建ててみる
description: pleromaというマストドンの軽量版みたいなものがあるらしいのでインスタンスを建ててみる。
date: 2017-11-13
slug: create-pleroma-instance
tags: 
  - "Web service"
  - "CentOS"
---
pleromaというマストドンの軽量版みたいなものがあるらしいのでインスタンスを建ててみる。


## やりかた

基本的には

[https://git.pleroma.social/pleroma/pleroma/wikis/Pleroma%E3%81%AE%E5%85%A5%E3%82%8C%E6%96%B9](https://git.pleroma.social/pleroma/pleroma/wikis/Pleroma%E3%81%AE%E5%85%A5%E3%82%8C%E6%96%B9)

ここをみて入れる。が、ubuntuかな？を想定しているみたいなので適宜読み替える。

### Erlang

まず、Erlang(プログラミング言語)を入れる

[https://packages.erlang-solutions.com/erlang/](https://packages.erlang-solutions.com/erlang/)

ここでrpmのリンクを取得。

```
wget リンク

yum localinstall ファイル名
```

### Elixir

次にElixir(Erlang VM上で動くプログラミング言語)を入れる

https://qiita.com/sonots/items/335a1cff003d70feff96

ここをみて、ちゃんとパスまで通す

※注意

そのままコピペすると古いバージョンが入ります（当たり前）最新版を確認しましょう

起動してみると(コマンドはiex)次のようなワーニングがでる

```
Warning: the VM is running with native name encoding of latin1 which may cause Elixir to malfunction as it expects utf8. Please ensure your locale is set to UTF-8 (which can be verified by running “locale” in your shell)
```

気持ち悪いので消したい

[https://qiita.com/masashi127/items/5344b3ab1b86053cf10e](https://qiita.com/masashi127/items/5344b3ab1b86053cf10e)

ここをみる

だが、これはcentos6用で /etc/sysconfig/i18nファイルがない。
 

[http://zero-config.com/centos/changelocale-002.html](http://zero-config.com/centos/changelocale-002.html)

ので、ここをみる

これでエラーが消えた。

### build-essential

つぎの

```
apt install build-essential
```

については

[http://blog.64p.org/entry/2013/07/22/142457](http://blog.64p.org/entry/2013/07/22/142457)

ここに

```
yum groupinstall "Development Tools"
yum install kernel-devel kernel-headers
```

すればいいとあるので従ってみる。私の場合はすべてインストール済みだった。

次はnginx, postgresql, letsencryptだ

### nginx

私の場合はインストール済みなので割愛。

nginxレポジトリを追加してインストールすればいけるはずだ。

[https://www.nginx.com/resources/wiki/start/topics/tutorials/install/](https://www.nginx.com/resources/wiki/start/topics/tutorials/install/)

このあたりを見ればいけるはず。

### letsencrypt

これも導入済みなので割愛。

公式サイトを見ればわかるはず。cronの設定を忘れずに。

cronに登録すべきは

```
systemctl stop nginx && certbot renew --force-renew && systemc
tl start nginx
```

かな。たぶん。–force-renewしないほうがいいという話もあるのでそこはご自分で

### ユーザー追加

まあ、もとの文書のとおりに

```
adduser pleroma

passwd pleroma
```

でパスワード設定だけしときます

### pleroma入れる

git cloneとcd, mix deps.get, exitします

git cloneのURLが間違っている。

正しいのは

https://git.pleroma.social/pleroma/pleroma.git

です。.gitが抜けてる。

mixとかいろいろするためにブランチ変更。

```
git checkout develop
```

ここで、mixが出来ない問題発生。

環境変数を手動で設定してたから消えちゃった

そこで /etc/profile の適切なところ(rootかそうでないかで分けてpathmungeしてるifのおわったあと)に

```
pathmunge /opt/elixir/bin after
```

を追加

もちろんパスはelixir入れたとこに適宜変えてください

で、

```
mix deps.get
```

意外とすぐ終わった。


### pleromaの設定

元の文書のとおりにやります

```
cp config/dev.exs config/dev.secret.exs
```

でdev.secret.exsの中身を書き換えて…

config.exsの中身も書き換えて

```
mix ecto.create && mix ecto.migrate
```

をして…できない。

あ、configディレクトリに入ってたからだ。上に戻ってcreateとmigrateします。

ちょっと時間がかかります。

migrateできない。

データベース作成権限がないと言われるのでpostgresユーザーで

```
ALTER ROLE pleroma WITH CREATEDB
```

して作成権限を与えます。

今度は

type “jsonb” does not exist

というエラー。どうやらこれ、postrgres9.4から追加されたようで、今入ってるのが9.2なので無いらしい。

ということでアップグレードしないといけない。

 

ので、いっかい消します

```
yum remove postgresql
```

### 新しいpostgresを入れる

せっかくなので10を入れます

https://www.postgresql.org/download/linux/redhat/

此処で自分に適したものをプルダウンで選択

私の場合は

```
yum install https://download.postgresql.org/pub/repos/yum/10/redhat/rhel-7-x86_64/pgdg-centos10-10-1.noarch.rpm

yum install postgresql10-server

yum install postgresql10-contrib

/usr/pgsql-10/bin/postgresql-10-setup initdb
```

で、pg_hba.confを書き換えます

```
/var/lib/pgsql/10/data/pg_hba.conf
```

host all all 127.0.0.1/32 trust 追記、 host all all 127.0.0.1/32 peer をコメントアウト

幾つか実行します

```
psql -U postgres -h localhost

alter role postgres with password 'password';
```

ついでなので最初からデータベース作成権限を付与します

```
CREATE ROLE pleroma WITH LOGIN CREATEDB PASSWORD 'password';
```

pg_hda.conf

さっきのtrustをmd5に

```
host all all 127.0.0.1/32 md5

local all all md5

local all all peer はコメントアウト

systemctl restart postgresql-10
```

再起動

```
psql -U pleroma -h localhost
```

ログインできました。

create extensionします

```
psql -U postgres -h localhost

CREATE EXTENSION citext;
```

pleromaのところに戻ります

```
mix ecto.create && mix ecto.migrate
```

今度は成功

### nginx設定

```
cp /home/pleroma/pleroma/installation/pleroma.nginx /etc/nginx/sites-enabled/pleroma.nginx
```

とありますがちょっと変えて

```
cp /home/pleroma/pleroma/installation/pleroma.nginx /etc/nginx/conf.d/pleroma.conf
```

とします。

pleroma.confを書き換えます。

nginx起動時にエラーが出たので

```
include snippets/well-known.conf;
```

をコメントアウトします

servcie登録します

[https://qiita.com/DQNEO/items/0b5d0bc5d3cf407cb7ff](https://qiita.com/DQNEO/items/0b5d0bc5d3cf407cb7ff)

によると/etc/systemd/system/配下に置くといいようなので

```
cp /home/pleroma/pleroma/installation/pleroma.service /etc/systemd/system/pleroma.service
```

します。一応内容を確認します。

elixirのmixの場所の指定が違うので

```
ExecStart=/opt/elixir/bin/mix phx.server
```

に書き換えます

pleromaを/home/pleroma以外に入れた場合ほかにも書き換えないといけないようです

で、

```
systemctl start pleroma
```

が、、、できない。

これはserviceコマンド実行時に/opt/elixir/binの環境変数が追加されていないから。

/etc/systemd/system/pleroma.serviceのExecStartの内容を書き換えます。

```
ExecStart=/bin/bash -c 'PATH=/opt/elixir/bin:$PATH exec /opt/elixir/bin/mix phx.server'
```

これで環境変数を追加できます。

書き換えたので

systemctl daemon-reload

と

systemctl start pleroma

します。/etc/systemd/system配下にserviceファイルを置いたのでenableは必要ないそうです。

で、規約をカスタムします

```
/home/plerome/pleroma/priv/static/static/terms-of-service.html
```

を編集します。ただのhtmlです。

とりあえず出来たかな。

ではまた。
