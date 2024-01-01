---
title: "tmux でセッション番号を自動で詰めて、エラーも出さない方法"
emoji: "🐪"
type: "tech"
topics: ["tmux", "dotfiles"]
published: true
---

tmux でセッションを作成する場合、デフォルトでは名称に自動で連番が付与されます。
複数のセッションを起動した後で不要になったセッションを削除した場合、数字が歯抜けになってしまうことがあるため、きれいに詰め直したいと思う方もいるのではないでしょうか。

意外と参考情報が少なく、自分の環境ではエラーも起きてしまったため解消方法を紹介します。

# 歯抜けになった番号を自動でrenumberする

## 連番ではなくなる現象

以下のように複数のセッションが起動しているとします。この場合は3つです。

```shell
$ tmux ls
0: 1 windows (created Tue Jan  2 02:49:57 2024)
1: 1 windows (created Tue Jan  2 02:50:05 2024)
2: 1 windows (created Tue Jan  2 02:50:12 2024)
```

次に、1番のセッションを削除して再度セッションの一覧を見ると、

```shell
$ tmux kill-session -t 1 && tmux ls
0: 1 windows (created Tue Jan  2 02:49:57 2024)
2: 1 windows (created Tue Jan  2 02:50:12 2024)
```

0 の次に 2 のセッションが残る形となってしまいました。
これを自動で0, 1, ... というように、自動で再び連番となるようにしたいと思います。

なお、最後に記載しますがウィンドウの場合はtmuxの機能だけで簡単に実現することができます。

## 自動でrenumberするスクリプト

[こちらのIssue](https://github.com/tmux/tmux/issues/937) に参考となるスクリプトがありました。
こちらを参考にしつつ、手元ではエラーも起きたため以下のようなコードで実現しました。起きたエラーの内容については後述します。

### 最終的なコード

シェルスクリプトを用意し、tmux のフックでそれを呼び出します。

まずはシェルスクリプトです。単純に数字のセッション名を抜き出し、forループでrenameしているだけです。

```bash:tmux-renumber-sessions.sh
#!/usr/bin/env bash

set -u

sessions=$(tmux ls -F '#S' - | grep '^[0-9]\+$' | sort)

new_number=0
for number in $sessions; do
  tmux rename -t $number $new_number
  ((new_number++))
done
```

次にフックで先程のスクリプトを呼び出すために `.tmux.conf` に以下のコードを記述します。
シェルスクリプトのパスはご自身の環境に合わせて変えてください。

```
set-hook -g session-created "run ~/.dotfiles/tmux-renumber-sessions.sh"
set-hook -g session-closed "run ~/.dotfiles/tmux-renumber-sessions.sh"
```

またシェルスクリプトには実行権限を付けるようにしてください。

### ポイント

現在のセッション情報などを見るために `tmux ls` コマンドを使うことはよくあると思います。
この出力は `-F` オプションを使うことでフォーマットすることが可能です。

以下 tmux のマニュアルにおける、FORMATSの記述を一部抜粋します。

> Certain commands accept the -F flag with a format argument.  This is a string which controls the output format of the command.  Format variables are enclosed in ‘#{’ and ‘}’, for example ‘#{session_name}’.  The possible variables are listed in the table below, or the name of a tmux option may be used for an option's value.  Some variables have a shorter alias such as ‘#S’; ‘##’ is replaced by a single ‘#’, ‘#,’ by a ‘,’ and ‘#}’ by a ‘}’.

詳細は別途マニュアルを参照していただくとして、`#S` という名前で session_name を出力することがわかります。
これによりウィンドウの情報などは除外して、セッション名のみを標準出力することが可能になります。

出力例：

```bash
$ tmux ls -F '#S'
0
2
```

## 手元の環境で起きたエラー

前述した、手元の環境で起きたエラーについてです。
私は普段MacとUbuntuの2つで作業を行っており、設定ファイルは dotfiles により管理しています。
今回のtmuxスクリプトは上述のIssueを参考に、Ubuntuで作成し問題なく動作していたのですが、Mac環境で動かしたところ起動時に以下のようなエラーが表示されてしまいました。

```shell
'/Users/user/.dotfiles/tmux-renumber-sessions.sh' returned 1
```

ほかにセッションがない状況で新たなセッションを作った際に、`tmux ls` がメッセージをエラー出力したことで表示されるようです。
なにかキーを押せば通常の画面に復帰するのですが、少し気持ちが悪いです。

この表示が出た時点でのBashスクリプトでは、以下のようにしてセッション一覧を取得していました。

```bash
sessions=$(tmux ls -F '#S' | grep '^[0-9]\+$' | sort)
```

分かりづらいですが、 **`tmux ls -F '#S'`** の部分が最終的なコードとは異なります。
以下のように標準出力に出すよう修正することでエラーが表示されないようにしました。

```diff
- sessions=$(tmux ls -F '#S' | grep '^[0-9]\+$' | sort)
+ sessions=$(tmux ls -F '#S' - | grep '^[0-9]\+$' | sort)
```

この事象が起こったのはMacの tmux 3.3a バージョンですが、tmuxのバージョン依存なのか、MacとLinuxの差によるものなのかまでは調査していないため不明ですが、今回はこのようにして修正することができました。

## ウィンドウの番号を自動で詰める場合

ウィンドウの番号を自動で詰め直す方法も一応書いておきます。
この場合は簡単です。tmux の機能があるため以下を `.tmux.conf` に書けばOKです。

```
set -g renumber-windows on
```

## おわりに

ニッチな需要かもしれませんが、意外と情報がなく困ったのでどなたかの助けになれば幸いです。
