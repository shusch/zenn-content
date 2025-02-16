---
title: "シェルスクリプトの curl にとりあえず付けておくべきオプション（＋たまに使うオプションのまとめ）"
emoji: "⛳"
type: "tech"
topics: ["shell", "bash", "cli", "linux"]
published: true
---

## はじめに

シェルスクリプト中で API からデータを取得する際など、 curl でどんなオプションを指定しておくか毎回悩んでしまうのでまとめておきます。

ついでに必要に応じて使うオプションもまとめておきます。

※シェルスクリプトに限らず CLI でも有用です

## とりあえず指定しておくオプション

基本的に以下のオプションはつけるように覚えておいて問題ないでしょう。

```sh
curl -fsSL <URL>
```

このオプション指定はとりあえず覚えておいて、必要に応じて他のオプションも付けるようにしておけば良いと思います。

## 各オプションの説明

### -f, --fail

http ステータスコードがエラーの場合にスクリプトを異常終了します。

curl はデフォルトだとサーバーがエラーのステータスコードを返した場合、サーバーが返したエラーのHTMLを表示し、スクリプト自体は正常終了を返します。

-f オプションを使うと、ステータスコードに応じてコマンドが異常終了してくれます（エラー 22 を返す）。
シェルスクリプトでは `set -e` と併せて使うのが良いです。

### -s, --silent

進捗状況とエラーメッセージを非表示にします。

スクリプトで実行するのであれば、進捗は邪魔なので非表示にしておきます。
ただしエラー時のメッセージは表示したいので、次の -S オプションを使います。

### -S, --show-error

-s オプションと併せて使い、エラー時のメッセージは表示するようにします。

### -L, --location

リダイレクトを有効にします。

指定しないとリダイレクトを辿ってくれません。

## 必要に応じて使うオプション

### -X, --request

HTTP メソッドを指定します。

```sh
curl -X POST https://example.com
curl -X PUT https://example.com
curl -X PATCH https://example.com
curl -X DELETE https://example.com
```

### -H, --header

ヘッダーを指定します。

```sh
curl -H 'Authorization: Bearer <token>' https://example.com
```

### -d, --data

指定したデータを送信します。

POST や PUT, PATCH などの指定と併せて使います。

```sh
curl -X PUT
     -H 'Content-Type: application/json' \
     -d '{"name": "John", "age": 20}' \
     https://example.com
```

### -o, --output

レスポンスを標準出力ではなく、指定したファイルに書き込みます。

```sh
curl -fsSL -o filename.tar.gz https://example.com/somefile.tar.gz
```

### -O, --remote-name

こちらは大文字の `O` （オー）です。

小文字の `o` と同じくファイル保存のオプションですが、こちらはリモート側のファイル名がそのままローカルのファイル名としてダウンロードされます。

特にファイル名を指定する必要がなければ、こっちを使えば良いです。

```sh
curl -fsSL -O https://example.com/somefile.tar.gz

# 最後のスラッシュ以後の `somefile.tar.gz` のファイル名で保存される
$ ls
somefile.tar.gz
```

単にダウンロードするだけなら特にオプションの要らない `wget` も使い勝手が良いです。

### -I, --head

HTTP ヘッダーのみを取得します。

### -b, --cookie

Cookie をサーバーに送信します。

Cookie は `name=value` 形式でも、次の -c オプションで保存したファイルを指定することもできます。

```sh
# 値を指定
curl -b 'name1=value1; name2=value2' https://example.com
curl -b 'name1=value1' -b 'name2=value2' https://example.com

# ファイルを指定
curl -b cookie.txt https://example.com
```

### -c, --cookie-jar

サーバーから Set-Cookie で返ってきた Cookie をファイルに保存します。

```sh
curl -c cookie.txt https://example.com
```

### -u, --user

認証に使うユーザー名とパスワードを指定します。

Basic 認証で使うことが多いです。

```sh
curl -u <user:password> https://example.com
```
