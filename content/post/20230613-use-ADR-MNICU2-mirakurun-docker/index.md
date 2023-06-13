---
title: ADR-MNICU2をdocker-mirakurun-epgstationでつかう
description: サンワサプライのカードリーダーであるADR-MNICU2をdocker-mirakurun-epgstationでつかう方法
date: 2023-06-13
slug: use-adr-mnicu2-mirakurun-docker
---
## 何が問題なのか
- B-casカードのカードリーダーとして[サンワサプライのADR-MNICU2](https://www.sanwa.co.jp/product/syohin?code=ADR-MNICU2)が使いたい。しかし、これは標準ドライバでは動作しないうえ、linuxドライバが提供されていない。
- mirakurunのイメージにドライバを入れこまないといけない。

## 解決
- このカードリーダーは[CIR125A](https://www.abcircle.co.jp/jp/product/21/CIR125A/)という製品のOEM品なのでそれのドライバを使えばいい。
- ドライバを入れたmirakurunのdockerイメージを作るDockerfileを作った。 [pinfort/mirakurun-docker-CIR115A](https://github.com/pinfort/mirakurun-docker-CIR115A)
- docker-mirakurun-epgstationを使うときにサンプルからコピーするdocker-compose.ymlを編集し、 chinachu/mirakurun:latest の代わりにpinfort/mirakurun-docker-cir115a:latest を使うようにする。
