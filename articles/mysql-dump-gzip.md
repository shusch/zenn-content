---
title: "MySQL で特定のテーブルを効率的にダンプして gzip 圧縮して出力する"
emoji: "🦈"
type: "tech"
topics: ["mysql", "shell"]
published: true
---

## はじめに

色々な実験を繰り返しているうちに、検証環境のDBデータに不整合が出ていることが判明して dump から復旧を行うことがありました。

とはいえ全てのテーブルが壊れていた訳ではなくデータ量も大きかったので、特定のテーブルだけのダンプデータをgzip圧縮して作成するシェルスクリプトを作成しました。

MySQLを使っていれば、他の環境でも流用できるスクリプトになっています。
（ただし InnoDB を想定しています）

## ダンプスクリプト

まずはスクリプトの全体です。

コメントを入れた2箇所（DB接続情報・テーブル名の指定）はダミーの値なので、環境に合わせて変更してください。

シェルスクリプトに実行権限を付けて実行すると、 `<データベース名>.sql.gz` というファイルが生成されるので、復元時はそのファイルをリストアします。

```sh
#!/usr/bin/env bash

set -euo pipefail

# 実際のデータベース情報に修正する
db_host="example.com"
db_user="root"
db_password="password"

declare -A target=(
  # キー `app` にデータベース名、値にテーブル名をスペース区切りで指定する
  # 複数のデータベースをダンプしたい場合は `app2` のようにキーバリューの組をその分だけ増やす
  ["app"]="prices products users"
  ["app2"]="admin_users order_histories purchase_histories"
)

# これより下は変更しなくてOK

for db_name in "${!target[@]}"; do
  tables="${target[$db_name]}"
  dump_file="${db_name}.sql"

  if [ -e "${dump_file}" ]; then
    rm "${dump_file}"
  fi
  if [ -e "${dump_file}.gz" ]; then
    rm "${dump_file}.gz"
  fi

  for table_name in ${tables}; do
    echo "Dumping ${db_name}.${table_name}..."
    mysqldump -h"$db_host" -u"$db_user" -p"$db_password" --quick --single-transaction "$db_name" "$table_name" >> "$dump_file"
  done

  gzip "${dump_file}"
  echo "Dump for ${db_name} completed and compressed."
done

echo "Completed!"
```

### ダンプスクリプトの実行

MySQLの接続情報はスクリプト中に定義しているので、実行時に引数は不要です。
実行権限だけ付けて実行すればOKです。

```sh
# 実行権限を付与
$ chmod +x dump.sh

# スクリプトを実行
$ ./dump.sh
```


## リストア

ダンプファイル `<db_name>.sql.gz` に対して、以下のコマンドを実行してください。
（user, host, データベース名はリストア先の情報を指定します）

```sh
zcat app.sql.gz | mysql -u<user> -h<host> -p <データベース名>
```

Linux環境であれば基本的に zcat が使えるはずですが、Macなどで zcat が使えない場合は zcat の部分を `gzcat` に置き換えてください。

```sh
# for Mac
gzcat app.sql.gz | mysql -u<user> -h<host> -p <データベース名>
```

## ざっくり説明

スクリプトについてざっくりと説明します。

## shebang

```sh
#!/usr/bin/env bash
```

Bashでスクリプトを実行するように指定しています。
`#!/bin/bash` で指定する場合もありますが、`/usr/bin/env` を使うと brew や apt などでユーザーがインストールしたBashがある場合、PATH定義に従ってそのBashを使うことができます。

```sh
set -euo pipefail
```

これは未定義の変数や処理の途中でエラーがあった場合にスクリプトを終了させるように指定しています。

## DB接続情報・ダンプ対象定義

ダンプしたいDBへの接続情報を与えてください。

```sh
db_host="example.com"
db_user="root"
db_password="password"
```

以下の部分にダンプを取得したい対象のデータベース名・テーブル名を定義してください。
キーにデータベース名、値にそのデータベースの中ダンプを取得したいテーブル名を指定します。

```sh
declare -A target=(
  ["app"]="ここにダンプ対象のテーブル名をスペース区切りで指定する"
  ["app2"]="hoge fuga piyo"
)
```

## dump 実行部分

このスクリプトではテーブルごとにダンプを取得しています。

```sh
  for table_name in ${tables}; do
    echo "Dumping ${db_name}.${table_name}..."
    mysqldump -h"$db_host" -u"$db_user" -p"$db_password" --quick --single-transaction "$db_name" "$table_name" >> "$dump_file"
  done
```

元々は以下のように、同じデータベースのテーブルは一度にダンプを取っていたのですが、テーブル数が多い場合に `Argument is too long` でエラーとなってしまったためテーブル毎にループさせています。

テーブル数が少なければ以下のように処理したほうが効率的ですが、汎用的に使えるようループをしています。

```sh
  # 元々はテーブルを一括でダンプしていた
  mysqldump -h"$db_host" -u"$db_user" -p"$db_password" --quick --single-transaction "$db_name" "$tables" > "$dump_file"
```

## mysqldump のオプション

mysqldump のオプションとして `--quick`, `--single-transaction` を指定します。

--quick オプションは処理がメモリ不足となるのを避けるために、クエリのキャッシュを無効化します。

詳細は以下の記事を参照してください。
https://zenn.dev/shuh/articles/mysql-option-quick

--single-transaction オプションはダンプデータの整合性を取るために指定します。
https://dev.mysql.com/doc/refman/8.0/ja/mysqldump.html#option_mysqldump_single-transaction

## おわりに

さすがに本番環境でダンプ・リストアが必要にはなっていないのですが、開発環境では色々な人が検証で触る都合上、リストア作業の必要が何度かありました。

一度書いておくと使い回せて便利なスクリプトだったので、リストア作業が必要になってしまった方はぜひ使ってみてください。
