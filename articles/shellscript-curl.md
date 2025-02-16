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

```
curl -fsSL <URL>
```

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

### -H, --header

ヘッダーを指定します。

```
curl -H 'Authorization: Bearer <token>' https://example.com
```

### -X, --request

HTTP メソッドを指定します。

```
curl -X POST https://example.com
curl -X PUT https://example.com
curl -X PATCH https://example.com
curl -X DELETE https://example.com
```

### -d, --data

指定したデータを送信します。

POST や PUT, PATCH などの指定と併せて使います。

```
curl -X PUT
     -H 'Content-Type: application/json' \
     -d '{"name": "John", "age": 20}' \
     https://example.com
```

### -o, --output

レスポンスを標準出力ではなく、指定したファイルに書き込みます。

ダウンロードするだけであれば `wget` が使われることも多いですが、 `wget` はデフォルトでインストールされていない場合もあり、 `curl` であればほぼ間違いなく入っています。

```
curl -fsSL -o filename.txt https://example.com
```

### -I, --head

HTTP ヘッダーのみを取得します。

### -b, --cookie

Cookie をサーバーに送信します。

Cookie は `name=value` 形式でも、次の -c オプションで保存したファイルを指定することもできます。

```
# 値を指定
curl -b 'name1=value1; name2=value2' https://example.com
curl -b 'name1=value1' -b 'name2=value2' https://example.com

# ファイルを指定
curl -b cookie.txt https://example.com
```

### -c, --cookie-jar

サーバーから Set-Cookie で返ってきた Cookie をファイルに保存します。

```
curl -c cookie.txt https://example.com
```

### -u, --user

認証に使うユーザー名とパスワードを指定します。

Basic 認証で使うことが多いです。

```
curl -u <user:password> https://example.com
```
