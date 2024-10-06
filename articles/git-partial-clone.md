---
title: "大きいGitリポジトリの clone が遅いなら Partial Clone を使おう"
emoji: "🍯"
type: "tech"
topics: ["git"]
published: true
---

## はじめに

OSSで古くからあるGitリポジトリをクローンする場合、履歴の数がとても多くクローンにかなり長い時間がかかることがあります。

OSSでなくとも、ファイル数やコミット数が多ければ同様になるでしょう。

もし通常のクローンしか使っていなければ Git の機能である Partial Clone を使ってクローンにかかる時間を短縮してみましょう！

## Partial Clone とは

パーシャルクローンとは、平たく言うとリモートリポジトリからダウンロードする情報を一部切り捨てることでデータ量を減らす方法です。

以下のようにして実行します。

```sh
# blobless clone
git clone --filter=blob:none <url>

# treeless clone
git clone --filter=tree:0 <url>
```

完全なクローンとは異なり情報を切り捨てていることから、当然デメリットも存在します。
しかし適切な条件で使用すれば、かなり有益な手法となるでしょう。

また情報の切り捨て方にも種類があり、パーシャルクローン以外にもシャロークローン (Shallow Clone) という方法が存在します。
ここではそれぞれの手法の違いと、主な用途について紹介します。

## クローンの種類

限定的なクローンを行う方法としては、主に以下の種類があります。

- パーシャルクローン
  - ブロブレスクローン (Blobless clone)
  - ツリーレスクローン (Treeless clone)
- シャロークローン

これらの違いを理解するためにはまず、Gitにおけるオブジェクトを知る必要があります。

## Gitのオブジェクト

Gitにおけるオブジェクトの詳細については、以下の記事がとても詳しくまとめられています。

https://zenn.dev/kaityo256/articles/objects_of_git

本記事でも、以下3つのGitオブジェクトについて、以後の説明がわかるよう簡潔に説明します。

- ブロブ (blob) オブジェクト
- コミット (commit) オブジェクト
- ツリー (tree) オブジェクト

## ブロブ (blob) オブジェクト

blobとは **B**inary **L**arge **OB**ject の略で、`git add` 実行時に `.git/objects` 配下へ作成されるものです。

ファイルの中身（にヘッダーを付与したデータ）を圧縮したオブジェクトであり、その SHA-1 ハッシュ値をファイル名としています。

つまり（コミット間の差分ではなく）**ファイルの実体そのものをオブジェクトとして保存している**、ということです。

:::message
たまにGitのコミットはファイルの差分を保存したものだと誤解されますが、blobを考えると誤った考えであることがわかります。
:::

## コミット (commit) オブジェクト

コミットオブジェクトは名前の通り、コミットの作成者・作成日時・コミットメッセージ などの情報が格納されています。

`git cat-file` コマンドを用いると、そのGitオブジェクトの中身を見ることができます。
これをコミットオブジェクトに使い、実際にコミット情報が保存されていることを確認してみましょう。

参考としてLinuxカーネルのGitリポジトリを使います。
以下は v6.8 (`e8f897f4afef0031fe618a8e94127a0934896aba`) のコミットオブジェクトです。

```
$ git cat-file -p e8f897f
tree 2976ac61587aa5acdfb6abbcf16e60e730287e98
parent fa4b851b4ad632dc673627f38a8a552547568a2c
author Linus Torvalds <torvalds@linux-foundation.org> 1710103089 -0700
committer Linus Torvalds <torvalds@linux-foundation.org> 1710103089 -0700

Linux 6.8
```

コミットの情報が含まれていることがわかりますね。

先頭行の tree が示すのは、次に説明する tree オブジェクトのIDになります。

## ツリー (tree) オブジェクト

blob オブジェクトはGitで管理しているファイル実体の情報は持ちますが、どのディレクトリにあるのか？の情報は持っていません。

そこでGit管理下にあるディレクトリ・ファイルの階層情報を持つのが tree オブジェクトです。

`git cat-file` コマンドで tree オブジェクトの中身を見てみましょう。
コミットオブジェクトで確認した、 `tree 2976ac61587aa5acdfb6abbcf16e60e730287e98` の tree オブジェクトを使ってみます。

```
$ git cat-file -p 2976ac6
100644 blob ccc9b93972a99eaca14b0cce1f50b86ab3c3947f    .clang-format
100644 blob 43967c6b20151ee126db08e24758e3c789bcb844    .cocciconfig
100644 blob 854773350cc5a92bf40312f8dce0f452166c19d6    .editorconfig
100644 blob c298bab3d3207fc5c6dd81d843b8a145ea8c655d    .get_maintainer.ignore
100644 blob 2325c529e1854545a1b611b72ad156bdbc4319cf    .gitattributes
100644 blob 689a4fa3f5477aa0fd46997eca5ee7a29cd78f8a    .gitignore
100644 blob bd9f1025ac44e0e289a6843de2c4497be2b76118    .mailmap
100644 blob 3de5cc497465c2722e160cb76188a8b287e48b7f    .rustfmt.toml
100644 blob a635a38ef9405fdfcfe97f3a435393c1e9cae971    COPYING
100644 blob df8d6946739f68655a8b077f0ebcc4bf4612944b    CREDITS
040000 tree a1a8eef9ce31ab1a7aa840d6159d52672e0c1f3c    Documentation
100644 blob 464b34a08f51ef9d2ae12e6902c92690b5dfa03b    Kbuild
100644 blob 745bc773f567067a85ce6574fb41ce80833247d9    Kconfig
040000 tree 3ec9c25a010a35c5e9f14f8e5dea7e63ad5e9a47    LICENSES
100644 blob 1aabf1c15bb30390f6286a291afdee9c93307ada    MAINTAINERS
100644 blob c7ee53f4bf044539bb783fbccd207b449a2dd880    Makefile
100644 blob 669ac7c32292798644b21dbb5a0dc657125f444d    README
040000 tree 8f518cd7f74aa3a8bd9a7934c185993636f8bbe7    arch
040000 tree 5c6297e65d621f00ed957d215290b4f54b46092a    block
040000 tree 463bbe458db1336006c1c6708f506bf1f83f10f9    certs
040000 tree 3c7f77d2881b5814e8246bc7c11cc68f257e18c2    crypto
040000 tree ab26b0576054020c9a58485887068e498630cafe    drivers
040000 tree c9483ae41d664b87563444ea3c4e965cfc9f72e8    fs
040000 tree 26e3e325520dc1613ef2d2622f6c9d14b4001e17    include
040000 tree 5e87fe55ab9f07646c7caa270594ff7151b638e1    init
040000 tree 4ef62764ad9490d308f88661b947c04b5421ce99    io_uring
040000 tree f6e448b44db272cc27436093dce119c20a671f50    ipc
040000 tree 61cd267702e2aad68b610d5efee1727d9ee9613f    kernel
040000 tree 42161b428d126cc781609a48a7c29c9de07450df    lib
040000 tree 4330f832d800e643acbd1ca690790eb9a434089e    mm
040000 tree 9ae481eb51655c17e9a0db9c20e753f2624bcd54    net
040000 tree e3753994fbfe6556b71b09d069a539d2156a3b8e    rust
040000 tree 010a4500b2e1e9bce9b8cde81e4344f50bc4518d    samples
040000 tree ea2d393dec9e96f7b09e756ffb62d4c8d5788d7e    scripts
040000 tree 100a40da11b6ce68bb15284d8ecbe978566f7f51    security
040000 tree 2b3ad962b102d52ee05159b883a0320db5010ed8    sound
040000 tree 87b3eccae77167d962bae40757d3dbcfb71173e9    tools
040000 tree 11baee12fc261e9e9a0c06edb72650113727e538    usr
040000 tree 0251d7a11843c0c47f0236860f9e2be8805a6ec1    virt
```

以下に v6.8 時点のソースコードがあります。
（並び順は違いますが）ファイルとディレクトリが一致していることが確認できます。

https://github.com/torvalds/linux/tree/v6.8

子階層のディレクトリは別の tree オブジェクトで表され、さらにその下は… として最下層のファイルまでの階層構造を作っています。

## Gitオブジェクトのまとめ

これまで説明したGitオブジェクトについてまとめると、それぞれ以下のようなものになります。

| オブジェクト | 説明 |
|--- | --- |
| commit | コミットの情報と、対応する tree オブジェクトを格納したもの |
| blob | ファイルを圧縮したもの |
| tree | ある階層のファイル・ディレクトリ一覧をまとめたもの |


ファイルが1文字でも変われば新しい blob が作られ、それを包む tree オブジェクトも新規に作られますし、コミット1つに対して commit オブジェクトも1つ作られます。

つまり、リポジトリにファイルの変更やコミット数が増えるにつれ、これらのオブジェクトも肥大化していくことがわかるでしょう。

## Partial Clone の使い方

前段がとても長くなりましたが、パーシャルクローンについて説明します。

最初に書いたように、パーシャルクローンにはブロブレスクローン (Blobless clone) とツリーレスクローン (Treeless clone) の2つがあります。

### Blobless Clone

ブロブレスクローンでは、すべての履歴から commit, tree オブジェクトを取得し、blob はチェックアウトしたコミットでのみ取得します。

blob はファイルそのものを圧縮したオブジェクトという性質上、データ量が大きくなります。
この方法では履歴上の blob をダウンロードしないため、clone にかかる時間をかなり短縮できます。

一方で commit, tree オブジェクトは clone 時にすべてダウンロードします。
そのため過去のコミット情報や `git log` など、ファイルの内容を必要としない操作は（追加のオブジェクトダウンロードが必要なく）軽量に動作させることが可能となります。

逆に、`git diff` や `git blame` などファイルの内容を知る必要がある操作の際は、blob のダウンロードが走るため、処理時間がフルクローン時よりも増えることになります。

#### 使い方

```sh
git clone --filter=blob:none <url>

# 例
git clone --filter=blob:none https://github.com/torvalds/linux.git
```

### Treeless Clone

ツリーレスクローンでは、commit オブジェクトのみ過去の履歴からダウンロードし、blob, tree オブジェクトは HEAD 時点のみダウンロードします。

一般的に、コミットが増えるにつれ tree オブジェクトはコミット数以上に増えていくことから、ツリーレスクローンはブロブレスクローンよりも更にクローンの時間を短縮することが可能です。

しかし tree と blob の両方が欠けていることから、過去のコミット点に切り替える際は、ブロブレスクローンよりも時間がかかることになります。

用途としては、そのコミット時点のファイルでのみビルド等ができれば良く、過去のコミット情報だけはローカルで参照もしたい場合といった限定的な使い道になるでしょう。

基本的にはブロブレスクローンを使うほうが良く、過去コミット情報すら要らない使い捨て用途であれば次に説明するシャロークローンを使ったほうが効率的です。

#### 使い方

```sh
git clone --filter=tree:0 <url>

# 例
git clone --filter=tree:0 https://github.com/torvalds/linux.git
```

## Shallow Clone

`--depth=<N>` オプションを指定すると、シャロークローンを使うことができます。
シャロークローンは、パーシャルクローンに比べて古くからGitの機能に存在するため、こちらのほうが有名かもしれません。

シャロークローンでは depth オプションに指定した数の分だけの履歴を取得します。
そのため、最初に取得したコミットより過去の履歴については、一切情報が抜け落ちてしまうことになります。

当然Gitコマンドを使ったほとんどの操作はできなくなるため注意が必要です。
あとで履歴にアクセスしようと思っても不可能になるため、基本的にはパーシャルクローンを使うことをおすすめします。

この方法は、そのコミット時点のファイルツリーでのみ操作ができれば良い、という場合には効率的にクローンができるため良い方法になるでしょう。
また時間課金のビルドツールなどを使っている際は、最も時間効率が良いことから最適な選択肢となります。

#### 使い方

```sh
git clone --depth=1 <url>

# 例
git clone --depth=1 https://github.com/torvalds/linux.git
```

## 比較表

紹介した3つの方法を比較して、それぞれの方法で取得するオブジェクトについてまとめます。

|  | commit | tree | blob |
| --- | :---: | :---: | :---: |
| **blobless clone** | ○ | ○ | × |
| **treeless clone** | ○ | × | × |
| **shallow clone** | × | × | × |

※HEAD についてはどの方式でも blob, tree, commit すべてを取得します

## まとめ

Gitのクローンで時間短縮を行う方法を3つ紹介しました。

特にブロブレスクローンは、この中でも最もバランスがよく大きなリポジトリのクローンに適しているでしょう。

歴史や規模が大きいリポジトリをクローンする際は、これらのオプションを活用してみてください。

## おまけ

おまけとして、記事で紹介した方法でLinuxカーネルのソースをクローンした時間を比較してみます。
細かく平均を取ったりはしていないので、あくまで参考として捉えてください。

クローンはGitHubのリポジトリから行いました。

実行時間は `time` コマンドで計測し、real の値を採用しています。
objects の列には、クローン時 `remote: Enumerating objects: XXX, done.` のように表示される、転送オブジェクトの数を表しています。

| クローン方式 | objects | time (real) |
| --- | ---: | ---: |
| full clone | 10,449,037 | 485.32s |
| blobless clone | 7,694,803 | 235.39s |
| treeless clone | 1,401,518 | 81.10s |
| shallow clone (depth 1) | 91,865 | 33.325s |

オプションを指定することで、かなりの時間短縮をできることが確認できましたね。
