---
title: "Amazon Linux 2023 に Supervisor をインストールして起動する"
emoji: "🐧"
type: "tech"
topics: ["aws", "amazonlinux2023", "linux", "supervisor"]
published: true
---

## はじめに

巷では着々と Amazon Linux 2023 への移行が進んでいるかと思います。
しかし Supervisor のインストールについて、 Amazon Linux 2 での方法をまとめたものはよくあるものの、Amazon Linux 2023 についてまとめたものはまだ多くありません。

というわけで Amazon Linux 2023 に Supervisor をインストールして起動するまでをまとめます。

※本記事で扱うのはインストール手順までで、詳細な設定ファイルのカスタマイズ方法までは触れません

## 環境

Amazon Linux 2023
Version 2023.6.20250107

## インストール

Amazon Linux 2 では yum 経由でインストールできますが、 Amazon Linux 2023 では Supervisor が RPM パッケージとして提供されていません。
（Version 6 時点）

以下のドキュメントに Amazon Linux 2023 Version 6 で利用できる全ての RPM パッケージが一覧となって記載されています。
https://docs.aws.amazon.com/ja_jp/linux/al2023/release-notes/all-packages-AL2023.6.html

Supervisor はここに記載されておらず `dnf search supervisor` などでもパッケージは見つかりません。
そのため pip 経由でのインストールを行っていきます。

### Python3, pip3 のインストール

まずは python3, pip3 がインストールされているかを確認します。

```
$ python3 --version
$ pip3 --version
```

python3 は確実にインストールされているはずですが、自分の環境では pip はインストールされていませんでした。

pip をインストールします。

```
sudo dnf install python3-pip
```

これで pip (pip3) がインストールされます。

### Supervisor のインストール

pip で Supervisor をインストールします。

```
sudo pip install supervisor
```

Supervisor の実行ファイルが存在していることを確認します。

```
$ which supervisord
/usr/local/bin/supervisord

$ which supervisorctl
/usr/local/bin/supervisorctl
```

## Supervisor の設定

設定ファイルを作成＆ /etc に設定ファイルを移動します。
Supervisor をインストールすると、デフォルトの設定ファイルを生成するコマンドも一緒に作成されるので、それを利用します。

設定ファイルを作成した後は、ファイルの所有者を変更したうえで /etc ディレクトリに設定ファイルを移動しておきます。

```
$ echo_supervisord_conf > ~/supervisord.conf
$ sudo chown root:root ~/supervisord.conf
$ sudo mv ~/supervisord.conf /etc/supervisord.conf
```

### 設定ファイルをインクルードできるようにする

実際に Supervisor でキューワーカーなどの設定を行う際は、管理したいプロセスごとにファイルを分けておいたほうが運用がしやすいです。
そのため、カスタマイズした設定をインクルードできるようにしておきます。

以下のように、 `/etc/supervisord.conf` の [include] の部分を変更してください。

```diff
169,170c169,170
< ;[include]
< ;files = relative/directory/*.ini
---
> [include]
> files = /etc/supervisord.d/*.ini
```

### 設定ファイルをカスタマイズ

以下のようにして、設定ファイルを作成してください。
（具体的な設定方法は他の方の記事を参照ください。ここは Amazon Linux 2 でも変わりません）

```
sudo vim /etc/supervisord.d/worker.ini
```

## systemd で起動できるようにする

ここまでやれば supervisord, supervisorctl コマンドで Supervisor を起動することは可能です。

しかし実運用上では systemd のサービスとして起動できるほうが便利なので、サービスに登録していきます。

### systemd のユニットファイルを作成

systemd で起動できるように、ユニットファイルを作成します。

なお、ユニットファイルが配置されるディレクトリとして `/usr/lib/systemd/system/` もありますが、こちらはパッケージ管理システムが提供するサービスのファイルが配置されるものです。

ユーザー自身で作成する場合は以下のように `/etc/systemd/system/` 配下に作成してください。
（/usr 配下のほうはパッケージのアップデートで上書きされる恐れもある）

```
sudo vim /etc/systemd/system/supervisord.service
```

ユニットファイルは以下の内容で作成します。
ExecStart で指定する supervisord のパスは `which supervisord` などで表示されるパスと合わせてください。

```
[Unit]
Description=Process Control System
After=rc-local.service nss-user-lookup.target

[Service]
Type=forking
ExecStart=/usr/local/bin/supervisord -c /etc/supervisord.conf

[Install]
WantedBy=multi-user.target
```

### systemd サービスとして Supervisor を起動する

ユニットファイルの変更を systemd に反映

```
sudo systemctl daemon-reload
```

最後に Supervisord の起動と自動起動の有効化をしておきます。

```
$ sudo systemctl start supervisord
$ sudo systemctl enable supervisord
```

`systemctl status supervisord` を実行して `Active: active (running)` のように表示されたらOKです！
