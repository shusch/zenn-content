---
title: "tmux でセッション番号を自動で詰め直す方法"
emoji: "🐪"
type: "tech"
topics: ["tmux", "dotfiles"]
published: true
---

tmux でセッションを作成する場合、デフォルトでは名称に自動で連番が付与されます。
複数のセッションを起動した後で不要になったセッションを削除した場合、数字が歯抜けになってしまうことがあるため、きれいに詰め直したいと思う方もいるのではないでしょうか。

意外と参考情報が少ないため、解消方法を紹介します。

# 歯抜けになった番号を自動で renumber する

## 連番ではなくなる現象

以下のように複数のセッションが起動しているとします。この場合は3つです。

```sh
$ tmux ls
0: 1 windows (created Tue Jan  2 02:49:57 2024)
1: 1 windows (created Tue Jan  2 02:50:05 2024)
2: 1 windows (created Tue Jan  2 02:50:12 2024)
```

次に、1番のセッションを削除して再度セッションの一覧を見ると、

```sh
$ tmux kill-session -t 1 && tmux ls
0: 1 windows (created Tue Jan  2 02:49:57 2024)
2: 1 windows (created Tue Jan  2 02:50:12 2024)
```

0 の次に 2 のセッションが残る形となってしまいました。
これを 0, 1, ... というように、自動で再び連番となるようにしたいと思います。

なお、最後に記載しますがウィンドウの場合は tmux の機能だけで簡単に実現することができます。

## 自動で renumber するスクリプト

[こちらの Issue](https://github.com/tmux/tmux/issues/937) に参考となるスクリプトがありました。
これを参考に、いくつか改良したコードが以下です。

### 改良版コード

シェルスクリプトを用意し、tmux のフックでそれを呼び出します。

数値のセッション名のみを grep し、それらを数値順にソートして rename していきます。

```bash:tmux-renumber-sessions.sh
#!/usr/bin/env bash

set -euo pipefail

new_num=0
while IFS= read -r num; do
  tmux rename -t "$num" "$new_num"
  new_num=$(( new_num + 1 ))
done < <(tmux ls -F '#S' | grep -E '^[0-9]+$' | sort -n)
```

次にフックで先程のスクリプトを呼び出すために `.tmux.conf` に以下のコードを記述します。
シェルスクリプトのパスはご自身の環境に合わせて変えてください。

```
set-hook -g session-created "run ~/.dotfiles/tmux-renumber-sessions.sh"
set-hook -g session-closed "run ~/.dotfiles/tmux-renumber-sessions.sh"
```

またシェルスクリプトには実行権限を付けてください。

```sh
chmod +x tmux-renumber-sessions.sh
```

### tmux ls の -F オプション

現在のセッション情報などを見るために `tmux ls` コマンドを使うことはよくあると思います。
この出力は `-F` オプションを使うことでフォーマットすることが可能です。

以下 tmux のマニュアルにおける、FORMATS の記述を一部抜粋します。

> Certain commands accept the -F flag with a format argument.  This is a string which controls the output format of the command.  Format variables are enclosed in ‘#{’ and ‘}’, for example ‘#{session_name}’.  The possible variables are listed in the table below, or the name of a tmux option may be used for an option's value.  Some variables have a shorter alias such as ‘#S’; ‘##’ is replaced by a single ‘#’, ‘#,’ by a ‘,’ and ‘#}’ by a ‘}’.

詳細は別途マニュアルを参照していただくとして、`#S` という名前で session_name を出力することがわかります。
これによりウィンドウの情報などは除外して、セッション名のみを標準出力することが可能になります。

出力例：

```bash
$ tmux ls -F '#S'
0
2
```

### 参考スクリプトからの改良点

- `sort` を `sort -n` に変更
  - `sort` はデフォルトで辞書順に並べ替えます。辞書順だとセッションが10個以上ある場合に `1, 10, 2` のような順になり、番号が正しく詰まらないため `-n` オプションを使い数値順で並べています
- `for` ループを `while IFS= read -r` に変更
  - 変数を展開してループする方式だと、単語分割や glob 展開の影響を受けるため、空白を含む値や glob 文字を含む値がある場合にループ対象が壊れてしまいます
  - これを避けるため、`while read` で 1 行ずつ読み込む形にしています
- プロセス置換 `< <(...)` を使用
  - パイプで `while` に渡すこともできますが、その場合ループがサブシェルで実行され、ループ内で更新した変数が外に反映されません
  - これを避けるためプロセス置換を使っています
- `set -euo pipefail` を追加
  - 未定義の変数を使った場合や、コマンド・パイプラインが失敗した場合にスクリプトを止める、安全策としての設定です
  - ただしプロセス置換内のコマンドは別プロセスで実行されるため、その失敗は検知されません
- カウンタの加算を `(( 算術式 ))` から算術展開 `$(( 算術式 ))` に変更
  - `(( 算術式 ))` は、算術式の評価結果が真（0 以外）なら終了ステータス 0（成功）、偽 (0) なら終了ステータス 1（失敗）を返します。本来は `if` などでの評価に使うものです
  - 後置インクリメント `((new_num++))` は評価結果が加算前の値になるため、初回（`new_num` が 0 のとき）は失敗扱いになります
  - そのため `(( 算術式 ))` を条件式ではなく代入目的で単独の文として使うと、`set -e` が有効な場合にスクリプトが止まることがあります
  - 算術展開 `$(( 算術式 ))` は式の結果を値として返すため、代入 `var=$(( ... ))` は評価結果が 0 でも失敗扱いになりません

なお、`while read` とプロセス置換は今回のスクリプトだと実害はありませんが、より汎用的で安全な書き方として採用しています。

## ウィンドウの番号を自動で詰める場合

補足ですが、ウィンドウの番号を自動で詰め直す方法も一応書いておきます。
この場合は簡単です。tmux の機能があるため以下を `.tmux.conf` に書けばOKです。

```
set -g renumber-windows on
```

## おわりに

ニッチな需要かもしれませんが、意外と情報がなく困ったのでどなたかの助けになれば幸いです。
