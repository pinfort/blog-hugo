---
title: Tsselect-ktをリリースしました
description: TsselectのKotlin移植版を作成しました
date: 2026-09-26
slug: tsselect-kt-release
---
TsselectのKotlin移植版をリリースしました。

## 概要
- ソースコード：[GitHub](https://github.com/pinfort/tsselect-kt)
- Mavenレポジトリ：[Maven Central](https://central.sonatype.com/artifact/me.pinfort/tsselect)

## 使い方
Tsselect-Ktは、CLIとしてもライブラリとしても利用できます。

### ライブラリとして使用する場合

JavaまたはKotlinプロジェクトから依存関係に追加して使用します。

Gradleの場合、以下のようにします。
```
implementation("me.pinfort:tsselect:1.1.0")
```

### CLIとして使用する場合
GitHubリリースの圧縮ファイルを展開し、binディレクトリ内の実行ファイルを実行します。

## 要件
### ライブラリとして使用する場合
Javaプロジェクトから使用する場合：JDK 17以上
Kotlinプロジェクトから使用する場合：Kotlin 2.0以上

### CLIとして使用する場合
JRE 25以上

## できること
CLIとして使用する場合、移植元本家の機能をそのまま実装しています。できることは同じです。
ライブラリとして使用する場合でもできることは同一ですが、独自のAPIによってプログラムから利用しやすいようになっています。具体的な使い方はREADMEをご覧ください。

## リンク
[オリジナル配布場所](https://www.marumo.ne.jp/junk/)

