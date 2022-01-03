---
title: MariaDBをQNAP NASに建てる
description: QNAP NASをつかって、MariaDBを立てました。
date: 2021-05-25
slug: mariadb-on-qnap-nas
tags:
  - Linux
---
## 用意するもの

- QNAPのNAS（TS 431-KX）

## やったこと

1. AppCenterからmariaDB10を入れる
2. mariaDBのアプリを開き、GUIで初期設定をする
2. AppcenterからPHP My Adminを入れる
3. QNAPのSSHをオンにする
4. PHPMyAdminの設定を変更する。
    
    mariaDB10のアプリを開くと、`ソケットへのパスが変更されており、設定ファイルの修正が必要です。`と書いてあるので変更する。File Stationなどから、アプリをインストールしたストレージ->Web->phpMyAdminフォルダを開き、config.inc.phpをダウンロードするなどし編集する。`$cfg['Servers'][$i]['AllowNoPassword'] = false;` の下に、 `$cfg['Servers'][$i]['socket'] = '/var/run/mariadb10.sock';` を書き加えて保存。アップロードして上書きする。phpMyAdminを開くと、admin, 最初に決めたパスワードでログインできる。

5. QNAPのシェルに入り、mariaDBの設定を変更する。デフォルトでは文字コードが軒並みlatin1になっているため。

    `/etc/my.cnf` が設定ファイルと思いきや、ちがう。phpMyAdminからサーバー->変数->baseDirを確認する。baseDir/etc/の下にあるmariadb.confを編集する。適宜編集前のファイルはバックアップを取っておく。
    clientとmysqlディレクティブに `default-character-set = utf8mb4`、mysqldディレクティブに 
    ```
    character_set_server=utf8mb4
    collation-server=utf8mb4_bin
    ```
    が設定されていることを確認して保存。
    mariaDBを再起動する。おそらく、`/etc/init.d/mariadb10.sh` にある。 `baseDir/etc/init.d/mariadb10.sh` へのsimLinkになっていることを確認して、`/etc/init.d/mariadb10.sh restart` 再起動してそうな文字が出てたらいい。phpMyAdminにログインしなおして、`show variables like "chara%";` のSQLを実行して、utf8mb4になってそうならOK。

6. phpMyAdminから、アプリ用のユーザーを作る。このとき、データベースも一緒に作るチェックボックスがあるので使うと便利。その場合、そのさらに下にある権限追加設定もすると便利。
7. コントロールパネル->Webサーバー->Webサーバーを有効にするにチェックして適用。
8. 作業用ＰＣなどからmariaDBにアクセスできるか確認してみる。出来たらOK
