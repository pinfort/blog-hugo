---
title: Laravel Mixが動いてくれない(Win10)
description: Windows 10環境において、Laravel Mixの実行でエラーが発生したので解決しました。
date: 2017-10-25
slug: laravel-mix-not-works-in-win10
tags:
  - "Laravel"
  - "Javascript"
---
Laravel MixがWindows 10で動いてくれない。

## したこと

 - PCにnodejsを入れる。
 - npm install --no-bin-linksした。

## 症状

npm run dev

したらエラーが出る。

## なおした

```bash
npm install --no-bin-links
npm run dev
```

もういっかいしてみる。

```
'cross-env' は、内部コマンドまたは外部コマンド、
操作可能なプログラムまたはバッチ ファイルとして認識されていません。
```

たりないものがあるみたい。

```bash
npm install --save-dev cross-env
```

いっこはかいけつした。

```bash
ERROR in ./node_modules/css-loader?省略./resources/assets/sass/app.scss
Module build failed: Error: ENOENT: no such file or directory, scandir '\path\to\project\node_modules\node-sass\vendor'
```

次のエラーはnode-sassのフォルダの中にvendorフォルダがないらしい。

同じ状態になってた人がいた。

[https://github.com/sass/node-sass/issues/1812](https://github.com/sass/node-sass/issues/1812)

```bash
npm rebuild node-sass
```

で解決した。
