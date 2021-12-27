---
title: PHPでfile_put_contentsが機能しない（Windows）
description: PHPでfile_put_contentsが機能しない事象が発生したのでメモ。
date: 2017-09-17
slug: php_file_put_contents_not_work_in_windows
tags:
  - "PHP"
---
PHPで`file_put_contents`が機能しない事象が発生したのでメモ。

## 事象
エラーは出ないのに、ファイルは保存されていない。

## 原因
保存ファイルパスの設定が原因。今回はファイル名に不正な文字（ファイル名に使用できない文字）が含まれていたため。

## 解決策
Windowsでファイル名に使用できる文字に置き換える。Windowsでファイル名に使用できない文字は以下。

¥ / : * ? ” < > |

 

## まとめ
保存ファイル名にファイル名として使用できない文字が含まれているとエラーが出ずに失敗する。

保存できなかったならエラーが出て欲しい。普通に保存できたような挙動をするのが謎。
