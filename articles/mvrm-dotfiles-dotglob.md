---
title: "mv や cp, rm コマンドで隠しファイルも対象にする方法4選"
emoji: "👻"
type: "tech"
topics: ["linux", "shell", "bash", "command"]
published: true
---

mv コマンドや cp コマンドで実行対象にワイルドカード ( `*` ) を使い、ディレクトリ配下の全ファイル・全ディレクトリを指定することがあります。

```sh
mv /path/from/* /path/to/
```

しかしワイルドカードを使うと、指定直下（上記の例だと `/path/from/` 直下）にある隠しファイル（ファイル名がドットから始まるファイル）は mv や cp, rm の対象になりません。

ワイルドカードを使うような状況では、隠しファイルも対象に含めたいのが普通だと思うので、いくつかの方法を紹介します。

以下に実験用に使えるリポジトリを用意しました。clone して from から to に移すよう色々とコマンドを試してみてください。
https://github.com/shusch/dotglob-playground

## 正規表現を使う方法

ドットから始まるファイルも含むよう正規表現で指定する方法です。
一番よく紹介されている方法な気がします。

```sh
mv /path/from/* /path/from/.[^\.]* /path/to/
```

個人的には正規表現が覚えられず、毎回調べてしまうので避けがちな方法です…

## cp コマンドを使う方法

`cp` コマンドを使うことで、他の方法より圧倒的に簡単な記述で実現することもできます。

ただし rm のような動作はできないので、`cp` と `mv` 限定です。
一番覚えやすいので個人的なおすすめ方法です。

#### `cp` を行いたい場合

```sh
cp -a /path/from/. /path/to/
```

#### `mv` を行いたい場合

```sh
cp -a /path/from/. /path/to/
rm -rf /path/from
```

この方法で `mv` を実現する場合 `/path/from` ディレクトリが消えるので、厳密に言えば `mv` と等価ではありません。
とはいえそこまで問題がある差異でもないと思われるので、この方法が最も簡単な指定方法でしょう。

## shopt コマンドを使う方法（Bashのみ）

`shopt` コマンドを使いシェルオプションを変更する方法です。

:::message
`shopt` はBashの組み込みコマンドのため、Zsh など他のシェルでは使用できません。
:::

デフォルトのシェル設定では、glob（ワイルドカードのパターン）に隠しファイルが含まれません。
そのためコマンドに `/path/from/*` を指定しても隠しファイルが対象から外れてしまっていました。

mv/cp/rm コマンド実行時に `shopt` で一時的に隠しファイルもワイルドカードのパターンにマッチするようオプションを変更します。

```sh
shopt -s dotglob # glob に隠しファイルを含むようシェルオプションを変更
mv /path/from/* /path/to/
shopt -u dotglob # シェルオプションをもとに戻す
```

以下のコマンドで、現在の設定値を確認できます。

```sh
shopt | grep dotglob
```

## find コマンドを使う方法

`find` コマンドで対象ディレクトリのファイル・ディレクトリを検索し、それを mv, cp, rm コマンドに受け渡して実行します。

```sh
# mv の場合
find /path/from -mindepth 1 -maxdepth 1 -exec mv -t to/ {} +

# cp の場合
find /path/from -mindepth 1 -maxdepth 1 -exec cp -rt to/ {} +

# rm の場合
find /path/from -mindepth 1 -maxdepth 1 -exec rm -rf {} +
```

- `-mindepth 1` は検索ディレクトリ（ここでは `/path/from` ）自身を検索結果に含めず、その中のアイテムのみを出力します
- `-maxdepth 1` は検索対象のディレクトリを第1階層までとして検索結果を出力します
- `{} +` は、findで見つかった複数のパスをまとめて `{}` の位置に展開します
  - `{} \;` を使うと、パスをまとめてではなく一つずつ分けて実行します
  - -exec の代わりに `xargs` をパイプでつないで使うこともできます。今回は引数をまとめて取れるので `-exec ... {} +` を使いました
- `mv`, `cp` の `-t` オプションは、対象元と対象先の順序を入れ替えることができます。通常はコピー/移動**先**を後ろに記述しますが、`-t` を指定すると前に書くことができます

#### `-exec` ではなく `xargs` を使う方法

```sh
# mv の場合
find /path/from -mindepth 1 -maxdepth 1 | xargs mv -t to/
# -I オプションを使う方法
find /path/from -mindepth 1 -maxdepth 1 | xargs -I{} mv {} to/

# cp の場合
find /path/from -mindepth 1 -maxdepth 1 | xargs cp -at to/
# -I オプションを使う方法
find /path/from -mindepth 1 -maxdepth 1 | xargs -I{} cp -a {} to/
```

## まとめ

mv や cp, rm コマンドで隠しファイルも対象にする方法4選を紹介しました。

個人的には [cpコマンドを使う方法](#cp-コマンドを使う方法) が一番覚えやすく使い勝手がいいかなと思っています。

[find コマンドを使う方法](#find-コマンドを使う方法) もこれを覚えておけば、-exec オプションや xargs コマンドなど様々な場面で応用が効くので、こちらもおすすめです。
