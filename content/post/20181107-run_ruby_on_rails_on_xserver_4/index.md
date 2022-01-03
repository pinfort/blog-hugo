---
title: XSERVERでRuby on Railsを動かす その4（結論編）
description: XSERVERでRuby on Railsを動かしてみた記録最終話です。すべての工程をこの記事にまとめていますので、前回までの記事を見る必要はありません。
date: 2018-11-07
slug: run_ruby_on_rails_on_xserver_4
tags:
  - "Ruby"
  - "Web service"
  - "XSERVER"
categories:
  - "run ruby on rails on xserver"
---
これは以前書いた　XSERVERでRuby on Railsを動かす　の結論をまとめて、わかりやすくしたものです。

#### 2019/04/27追記

この記事にある方法でYou are on Rails!のまま放置していたサイトがあるのですが、予想していた通り、いつの間にかプロセスが殺されていました。cronで回したりして無理やり動かすことはできるでしょうが、さすがに怒られそうなのでそこまではやりません。つまり、**XSERVERでRuby on Railsを動かすことはできるが、本番環境として長時間動かすことはできない** ということであり、動かすことができるとしても **ごく短時間の動作テスト程度にとどまるだろう** ということです。Ruby on Railsの本番環境としては、おとなしくVPSなどを使いましょう。

### なんといってもまずはRuby

Railsを動かすにはいわんやRubyが必要です。XserverにもRubyは入っているのですが、古すぎてRailsは動きません。そこで、新しいRubyを入れます。

でも、Xserverではyumが使えないので、ソースからコンパイルします。CentOS7、64bit環境で対応可能なので、頑張って同じ環境を用意してください。

また、インストール先は/home/user_name以下のどこでもいいのですが、ここでは/home/user_name/railsinstaller/rubyにインストールすることにします。

Rubyのコンパイルに必要なものは事前にインストールが必要です。

#### 必要なもの
- [最新安定版Ruby](https://cache.ruby-lang.org/pub/ruby/2.5/ruby-2.5.3.tar.gz)（2018/11/06時点2.5.3）

#### コマンド例
##### ローカルでの作業(CentOS7、64bit環境で行ってください。)

コンパイルに必要なものです。
```
yum -y install gcc zlib-devel openssl-devel readline-devel libffi-devel
```

持ってきて解凍します。
```
wget https://cache.ruby-lang.org/pub/ruby/2.5/ruby-2.5.3.tar.gz
tar xzvf ruby-2.5.3.tar.gz
cd ruby-2.5.3.tar.gz
```

インストール先を指定してconfigureします。user_nameはあなたのXserverのサーバーIDに変えてください。また、インストール先を自由に変更しても構いません。--enable-sharedは、つけないとダメです。
```
./configure --prefix=/home/user_name/railsinstaller/ruby --enable-shared
make
cd ../
tar -zcvf ruby-compiled.tar.gz ruby-2.5.3
```

ruby-compiled.tar.gzを何とかしてxserverの任意の場所にupload出来れば、CentOSでの作業は終了です。これ以降はすべてXserver側で作業を行います。

##### Xserver側の作業

作業用にディレクトリを作成します。名前は任意で構いません。
```
mkdir ~/sysad
cd ~/sysad
```

tar.gzファイルを作業ディレクトリに移します。
```
mv hogehoge ~/sysad/
```

解凍して
```
tar xzvf ruby-compiled.tar.gz
cd ruby-2.5.3
```

インストールを行います。
```
make install
```

インストール出来たら、パスを通します。
```
echo 'export PATH="$HOME/railsinstaller/ruby:$HOME/railsinstaller/ruby/bin:$PATH"' >> ~/.bash_profile
```

今回はすぐ確認したいのでbash_profileの更新を明示的に反映します。
```
source ~/.bash_profile
ruby -v
```

もともとは
```
ruby 2.0.0p648 (2015-12-16) [x86_64-linux]
```

だったと思うのですが、これが
```
ruby 2.5.3p105 (2018-10-18 revision 65156) [x86_64-linux]
```

こう変わっていれば成功です。

### Railsが依存しているNokogiriに必要なもののインストール
railsは様々なgemに依存しています。しかし、gem install railsではうまくいってくれません。まずは、一番ややこしいnokogiriというgemの依存を解決します。

#### 必要なもの
- [gmp(2018/11/07時点で最新は6.1.2)](https://ftp.gnu.org/gnu/gmp/gmp-6.1.2.tar.xz)

#### コマンド例
##### GMPのインストール
もってきます。
```
wget https://ftp.gnu.org/gnu/gmp/gmp-6.1.2.tar.xz
```

解凍します。
```
tar Jxvf gmp-6.1.2.tar.xz
cd gmp-6.1.2
```

configureします。インストール先は任意です。
```
./configure --prefix=/home/user_name/railsinstaller/gmp
```

コンパイルして入れます。
```
make
make install
```

### Nokogiri本体のインストール
依存しているGMPがインストールできたので、Nokogiri本体をインストールしていきます。しかし、これも一筋縄ではいきません。

#### 注意すること
- gmpを/home/user_name/railsinstaller/gmpにインストールしたので、gem install時にオプションでこのディレクトリを教えてあげないといけません。
- デフォルトのリンカだと、/usr/bin/ld: 認識できないオプション '--compress-debug-sections=zlib' ですというエラーが出てインストールできないので、リンカをGoldというやつに置き換える必要があります。

#### コマンド例
まず、リンカをgoldに置き換えます。私の環境では、~/binにパスが通っていたので、このディレクトリを作成し行いましたが、パスが通っていればどこでも構いません。
```
mkdir ~/bin
cd ~/bin
ln -s /usr/bin/ld.gold ld
which ld
```

ldの場所が、~/bin/ldになっていれば成功です。これが出来たら、いよいよ本体をインストールしていきます。
```
gem install nokogiri -- --with-ldflags='-L/home/user_name/railsinstaller/gmp/lib'
```

たくさんログが出て、時間がかかりますが、最後の行に1 gem installedと出れば成功です。

### nio4rのインストール
これも、gmpに依存しているので、gmpの場所を教えてあげます。

#### コマンド例
```
gem install nio4r -- --with-ldflags='-L/home/user_name/railsinstaller/gmp/lib'
```

### websocket-driverのインストール
同じで、gmpに依存しています。

#### コマンド例
```
gem install websocket-driver -- --with-ldflags='-L/home/user_name/railsinstaller/gmp/lib'
```

### Railsのインストール
ついに、railsをインストールする時が来ました。もし、途中で失敗したときは、どのgemのインストールでこけたのか確認し、そのgemのinstallを試してみましょう。たぶん、--with-ldflagsを追加してインストールしてやればいけると思います。

#### コマンド例
```
gem install rails
```

### bindexのインストール
railsは入りましたが、rails newするにはもう少し頑張る必要があります。これもgmpに依存しているので同じ対応をします。

#### コマンド例
```
gem install bindex -- --with-ldflags='-L/home/user_name/railsinstaller/gmp/lib'
```

### msgpackのインストール
同じで、gmp依存です。

#### コマンド例
```
gem install msgpack -- --with-ldflags='-L/home/user_name/railsinstaller/gmp/lib'
```

### bootsnapのインストール
同じで、gmp依存です。

#### コマンド例
```
gem install bootsnap -- --with-ldflags='-L/home/user_name/railsinstaller/gmp/lib'
```

### byebugのインストール
同じで、gmp依存です。

#### コマンド例
```
gem install byebug -- --with-ldflags='-L/home/user_name/railsinstaller/gmp/lib'
```

### ffiのインストール
同じで、gmp依存です。

#### コマンド例
```
gem install ffi -- --with-ldflags='-L/home/user_name/railsinstaller/gmp/lib'
```

### pumaのインストール
同じで、gmp依存です。

#### コマンド例
```
gem install puma -- --with-ldflags='-L/home/user_name/railsinstaller/gmp/lib'
```

### mysql2のインストール
xserverでは、mysqlが提供されていますので、デフォルトのsqliteでなくmysqlを使うことにします。その為にはmysql2が必要なのですが、mysql-develに依存していますので、これをインストールしていきます。なお、sqliteを使う場合は、mysql-develが必要ない代わりにsqlite-develが必要になります。頑張ってください。

#### 必要なもの
- [mysqlのコンパイル済みパッケージ](https://downloads.mariadb.org/mariadb/)

上記リンクからダウンロードができます。執筆時点で最新は10.3.10なのでこれを使います。多くのパッケージが並んでいますが、mariadb-10.3.10-linux-x86_64.tar.gzを選びます。

#### コマンド例
```
wget hogehoge
```

index.html?serveというよくわからない名前でダウンロードされたので名前を変更しておきます。

```
mv index.html?serve mariadb-10.3.10-linux-x86_64.tar.gz
```

解凍します。
```
tar xzvf mariadb-10.3.10-linux-x86_64.tar.gz
```

これはコンパイル済みなので置くだけでいいです。
```
mv mariadb-10.3.10-linux-x86_64 ~/railsinstaller/mysql
```

mysql-develがおけたので、gemをインストールします。このgemはgmpにも依存しているので二つのパスを教えてあげます。
```
gem install mysql2 -- --with-ldflags='-L/home/user_name/railsinstaller/gmp/lib' --with-mysql-dir=/home/user_name/railsinstaller/mysql
```

### rails new
これで、rails newができるはずです。出来なかったらbundle installしてみて。あと、mysqlを使うためのオプションも忘れないで。

#### コマンド例
```
mkdir ~/rails
cd ~/rails
rails new sample -d mysql
```

インストールできたので、起動できるか試します。
```
cd sample
./bin/rails s
```

### 外から見えるようにする
このままでは、外から見えないので、外から見えるようにします。以下のリンクからphpファイルをダウンロードしておいてください。また、railsのポートはデフォルトの3000でなく、適当なポートにすることを強く推奨します。

#### 必要なもの
- [自分の環境に合わせて編集したphpファイル](https://gist.github.com/pinfort/7bdcb0047a2ade84e591ddb5b1126f9a)

#### コマンド例
```
mkdir ~/railsinstaller/php
cd ~/railsinstaller/php
```

php5系ではうまくいかなかったので7系でやります。
```
curl -sS https://getcomposer.org/installer | /usr/bin/php7.2
/usr/bin/php7.2 composer.phar require guzzlehttp/guzzle
cd ~/公開したいドメイン/pubic_html
```

編集したphpファイルをindex.phpとして保存。
```
cd ~/rails/sample
```

railsを起動する前に、database.ymlにmysqlの設定をします。rails用のDBを作っていない人は、Xserverのサーバーパネルから作成できますのでしておいてください。ユーザーを割り当てるのも忘れずに。
```
cd config
vi database.yml
```

developmentディレクティブを以下のように書き換えます。hostは、サーバーパネル、mysql設定の下のほうに書いてあります。
```
development: 
  <<: *default
  database: user_name_railstest
  username: user_name_railsdb
  password: password
  host: mysqlxxxx.xserver.jp
```

設定はできました。あとは起動するだけです。
ここで、railsがターミナルを閉じた後も動くようにしないといけません。
```
nohup ./bin/rails s -p PHPファイルで指定したポート &
```

これで、index.phpがおいてあるドメインにアクセスすればRailsが見えると思います。

**Yay! You’re on Rails!**
