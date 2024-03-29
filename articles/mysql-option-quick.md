---
title: "mysqlコマンドからCSV出力しようとしたら空ファイルになったときの対処法"
emoji: "🦌"
type: "tech"
topics: ["mysql", "database"]
published: true
---

## はじめに

DBのデータでSELECTした結果をCSVに保存したい状況はよくあると思います。
具体的には、以下のようにユーザ一覧を取得するようなものです。

```
$ mysql -u user -h example.com -p -e 'SELECT * FROM users' > out.csv
```

この例では、example.com にデータベースが存在するとして `SELECT * FROM users;` の出力をローカルのファイルに保存しようとしています。
（この例そのままだとタブ区切りで出力されますが…）

同様にして、かなり大容量（100万レコード以上）のデータに対してCSV保存しようとしたところ、出力ファイルが空になってしまったため解消方法を紹介します。

## 結論

`--quick` または `-q` オプションを追加する。

```
$ mysql -u user -h example.com -p --quick -e 'SELECT * FROM users' > out.csv
```

## なぜこれで解消するか

`mysql` コマンドからSQLの実行結果を出力した場合、デフォルトでは出力結果がメモリにキャッシュされます。

今回のケースでは、空きメモリを超えた容量のデータを保存しようとしていたため、メモリ不足となり出力が空となってしまいました。

実際にCSVへの出力コマンドを実行中、以下のようにsarコマンドでメモリ使用量を監視してみたところ、みるみる空きメモリ(kbmemfree) が減っていき、ちょうど空きがなくなるタイミングでコマンドも終了することが確認できました。

```
$ sar -r 1
```

`--quick` オプションを使用すると、クエリ結果をキャッシュせず、逐次レコードを出力します[^1]。
このためメモリ不足に陥ることなく、出力結果をすべてファイルに保存することができました。

## まとめ

今回は大容量データをメモリ不足にならずにファイル保存する方法を紹介しました。
同じようにメモリが潤沢ではないサーバから大規模データを保存する際は `--quick` オプションで解消する可能性があります。

また、GUIのないサーバ上でメモリやCPU監視をする際には sar コマンドが便利です。覚えておくと困ったときに役に立つでしょう。

[^1]: https://dev.mysql.com/doc/refman/8.0/ja/mysql-command-options.html#option_mysql_quick
