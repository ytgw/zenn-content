---
title: "Go言語でSQLite3を使う"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Go, SQLite3]
published: true
---

## 概要
Go言語でのDB操作を試してみたので、紹介します。
RDBMSならどれでも良かったので、気軽に使えそうなSQLite3を今回試しました。
作成したソースコード等は[GitHubのリポジトリ](https://github.com/ytgw/go-hello-world/tree/main/sqlite)に置いています。

## 初期設定

初期設定として、下記コマンドを実行し、モジュール管理ファイルを新規作成します。

```bash
# モジュール管理の初期化(sqliteはモジュール名?なので任意)
go mod init sqlite
```

## SQLite用のドライバーのインストール

GoでDBを扱う場合SQL database driversを入れる必要があります。
ドライバーの一覧は[公式のWiki](https://github.com/golang/go/wiki/SQLDrivers)に記載されています。
今回はこの中から[mattn/go-sqlite3](https://github.com/mattn/go-sqlite3)を選択しました。
理由は確認時点で「both included in and pass the compatibility test suite at https://github.com/bradfitz/go-sql-test .」と記載がある唯一のSQLite3用ドライバーだったからです。
下記コマンドでインストールします。

```bash
# SQLiteのドライバのインストール
go get github.com/mattn/go-sqlite3
```

## コーディング

[mattn/go-sqlite3にある例](https://github.com/mattn/go-sqlite3/blob/master/_example/simple/simple.go)をそのまま使いました。
シンプルな内容で量も100行程度です。
Go言語にかかわらず、SQLをプログラムから使ったことがある人なら理解できると思います。


## ビルドと実行

```bash
go build sqlite.go

./sqlite
```

