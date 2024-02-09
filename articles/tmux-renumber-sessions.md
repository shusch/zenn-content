---
title: "tmux でセッション番号を自動で詰め直す方法"
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
ただ、このままだと手元の環境ではエラーが起きてしまったので少し修正したコードで実現しました。正常動作したコードは以下です。

同様にエラーとなった方向けに、エラーの内容については後述します。

### 最終的なコード

シェルスクリプトを用意し、tmux のフックでそれを呼び出します。

まずはシェルスクリプトです。単純に数字のセッション名を抜き出し、forループでrenameしているだけです。

```bash:tmux-renumber-sessions.sh
#!/usr/bin/env bash

set -u

sessions=$(tmux ls -F '#S' | grep '^[0-9]\+$' | sort)

new_number=0
for number in $sessions; do
  tmux rename -t "$number" $new_number
  ((new_number+=1))
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

動作させるだけであれば上述のコードを使ってもらえれば問題ありませんが、同様のエラーがあった方向けに詳細を記載します。

私は普段MacとUbuntuの2つで作業を行っており、設定ファイルは dotfiles により管理しています。
今回のtmuxスクリプトは、上述のIssueのコードにてUbuntu上では問題なく動作していたのですが、Mac環境で動かしたところTmux起動時に以下のようなエラーが表示されてしまいました。

```shell
'/Users/user/.dotfiles/tmux-renumber-sessions.sh' returned 1
```

tmuxのhookスクリプトの終了ステータスが「失敗」を示す `1` を返したために、tmux側でエラーとして表示されるようです。
なにかキーを押せば通常の画面に復帰するのですが、少し気持ちが悪いです。

## 原因の特定

先程のhookを使わない状態では起きていないエラーなので、hookスクリプトが原因なことは明らかでした。
直前に実行したコマンドの終了ステータスは `echo $?` で取得できるため、スクリプトの各行で返している終了ステータスを一つずつチェックしたところfor文の中で `new_number` に +1 している箇所が原因であることがわかりました。

```bash
  ((new_number++))
```

この箇所で何故か終了ステータスが 1 を返していたため、tmux起動時にメッセージが出るようになってしまっていたようでした。

「最終的なコード」で以下の部分を変更することで、特にエラーなくTmuxが起動できるようになりました。

```diff
-  ((new_number++))
+  ((new_number+=1))
```

この事象はUbuntu 22.04 環境では起きず、macOS 14 でのみ発生しました。Bashのバージョンによるものなのかなど根本的な原因までは不明ですが、今回はこのようにして修正することができました。

## ウィンドウの番号を自動で詰める場合

補足ですが、ウィンドウの番号を自動で詰め直す方法も一応書いておきます。
この場合は簡単です。tmux の機能があるため以下を `.tmux.conf` に書けばOKです。

```
set -g renumber-windows on
```

## おわりに

ニッチな需要かもしれませんが、意外と情報がなく困ったのでどなたかの助けになれば幸いです。
