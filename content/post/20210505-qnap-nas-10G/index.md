---
title: QNAP 10G NAS構築記録
description: QNAP TS-431KXを購入し、セットアップを行いました。その記録です。
date: 2021-05-05
slug: qnap-nas-10G
tags:
  - network
---
この度、QNAPの10G対応NASを購入したので記録として記します。

## 材料
|名称|価格|購入場所|
|--|--|--|
|[QNAP TS-431KX](https://www.qnap.com/ja-jp/product/ts-431kx)|52,500|ECjoy|
|MN07ACA12T*4|三万円弱/個くらい？|パソコン工房など|
|[DDR3L-1600 4G*2](https://www.amazon.co.jp/gp/product/B00JPXFSAK)（ただしこれはミスでした）|4,309|Amazon|
|10G DAC|家に落ちてた|fs.com|

## 組み立て
### NASのメモリ増設
購入したNASは、メモリが2Gしかなく心もとないので、4GB\*2のメモリを別途購入し、増設することとしました。しかし、ここで問題が発生。QNAPのNASは、メモリスロットが二つあることが多いのですが、このNASは一つだけでした。そのため、4GB\*2で8GBとしようとしたところ、4GBしか取付できませんでした。今度いつか、8GBメモリを購入して指しなおします。
メモリの取り付けには、一度ケースを丸々外す必要がありますが、ドライバでねじを外せば、後はスライドするだけで簡単でした。

### HDDの用意、設置
用意といっても、シールを貼るだけです。RAIDを組むので、ばらしたときに場所を入れ替えてしまっては大変です。そのため、HDD本体と、外ケースのそれぞれに、わかるようにシールを貼ります。いいシールがないか探したところ、ちょうどGREEさんのシールがあったので利用させてもらいました。最初、GREEの四文字で四つのHDDに貼り分けようとしていたのは内緒です。もちろん、三文字目と四文字目が同じEなので、それでは見分けがつきません。Eの代わりに、GREEの六角形マークを一文字として利用することで解決しました。

{{< figure src="assets/thumbnail/qnapnas_hdd.jpg" link="assets/qnapnas_hdd.jpg" title="qnapnas_hdd" height="340">}}

### 電源、ネットワークの接続
変わったことはなく、同梱されている電源を接続すること。また、10G用のケーブルは同梱されていないので（RJ45のLANケーブルは同梱されていました）、どのご家庭にも落ちている任意のSFP+ケーブルを用意し、これまたどのご家庭にも余っている10GスイッチのSFP+コネクタに接続するだけです。特に問題なくリンクアップもしました。

{{< figure src="assets/thumbnail/10Gdac.jpg" link="assets/10Gdac.jpg" title="10GDAC" height="340">}}

### 初期設定
QNAPのNASようにソフトウエアもあるようですが、単純にip直打ちでも接続できるようです。ユーザー名はadmin、初期パスワードはプライマリネットワークカードのMACアドレスのようですね。ログインができたら、ストレージプールの作成など、必要な手順を画面に従って行います。丁寧なチュートリアルがあるので、そう迷うことはなさそうです。今回は4本の12TBのHDDでraid6を構築しました。

{{< figure src="assets/thumbnail/storages.png" link="assets/storages.png" title="storages" height="340">}}

### ネットワーク速度の確認
さて、せっかく10GのNASを構築したので、速度を確認していたいところです。実際10Gネットワークで接続されたPCからベンチマークを行ったところ、読み込みでは約5Gbpsを記録するなど、素晴らしい成果を残してくれました。

{{< figure src="assets/thumbnail/network-bench.png" link="assets/network-bench.png" title="network-bench" height="340">}}
{{< figure src="assets/thumbnail/network-speed.png" link="assets/network-speed.png" title="network-speed" height="340">}}

### 終わりに
今回すばらしいストレージ環境を構築することができました。これをベースに、さらに環境を充実させていきたいと考えています。
