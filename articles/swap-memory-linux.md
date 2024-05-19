---
title: "Swapメモリの作成・削除"
emoji: "🍵"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["linux", "unix", "bash", "aws", "ec2"]
published: true
---

サーバ等のLinux環境でメモリ不足で困ったときの対応としてSwapメモリを作成する方法をまとめておきます。
一時的な対処だった場合に、作成したSwapメモリを削除してクリーンに戻す方法も記載します。

メモリ不足になっている状態自体は好ましいことではないので、本当はサーバ自体のメモリを増設するべきですが、
- オンプレ等で簡単にメモリの増設ができない😵
- AWS EC2 などのクラウドインスタンスでも、常時稼働しているサーバで冗長構成等もないので気軽に停止してスペックアップできない🤢
- メモリを増やすとなると費用がかかるし、各所に申請等しなくてめんどくさいので、ちゃちゃっと対処したい😈
- その他 ~~インフラ周りを触るのはめんどくさいので~~ とりあえず対応したい😇

こんな場合にSwapメモリを使って乗り切りたい場面はあると思います。

もちろん、正当な理由？でSwapメモリが必要な場合も使える手順です。

## Swapメモリの作成

### スワップファイルの作成

ここでは仮に1GBのスワップ領域を作成するとします。
スワップファイルのパーミッションも適切に設定しておきます。

```sh
sudo dd if=/dev/zero of=/swapfile bs=1M count=1024
sudo chmod 600 /swapfile
```

bsの値 × countの値がスワップ領域のサイズとなるので、例えば `bs=4M count=1024` とすれば4GBのスワップメモリが作成されることになります。

### スワップ領域の有効化

```sh
sudo mkswap /swapfile
sudo swapon /swapfile
```

### スワップ領域の確認

スワップ領域を確認します。

```sh
sudo swapon -s
```

swapメモリが有効になっていることが確認できます。

```sh
$ free -h
          total    used    free   shared   buff/cache   available
Mem:       1.9G    1.4G    243M      33M         312M        376M
Swap:      1.0G      0B    1.0G
```

### 再起動してもSwap領域が有効になるようにする

このままだと、サーバを再起動した場合Swap領域が消えてしまいます。
再起動後もスワップが有効とする必要があれば、追加で設定をします。

```sh
sudo vi /etc/fstab
```

ファイルに以下の行を追記して保存します。

```
/swapfile swap swap defaults 0 0
```

## Swapメモリの削除

不要になったスワップ領域は削除します。

```sh
sudo swapoff /swapfile
sudo rm /swapfile
```
