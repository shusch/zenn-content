---
title: "Linux/Mac CPUのコア数・スレッド数を調べるコマンド"
emoji: "🧲"
type: "tech"
topics: ["linux", "mac", "shell", "cpu"]
published: true
---

make の並列オプション指定など、コマンドを用いてCPUのコア数（物理コア数）、スレッド数（論理コア数）を取得したい場合があります。

Linux/Mac でコマンドが違い、いつもわからなくなるのでまとめておきます。


## 検証環境

Linux: AMD Ryzen 5 2600 Six-Core Processor
Mac: Apple M1


## コア数（物理コア数）を出力する


### /proc/cpuinfo を使う (Linux)

/proc/cpuinfo の出力を grep します。

```sh
$ grep core.id /proc/cpuinfo | sort -u | wc -l
6
```


### sysctl コマンドを使う (Mac)

```sh
$ sysctl -n hw.physicalcpu_max
8
```


## スレッド数（論理コア数）を出力する


### nproc コマンドを使う (主に Linux)

```sh
$ nproc --all
12
```

`--all` コマンドを指定すると、`taskset` コマンドのようにCPUの割り当てを制限した上で実行しても、マシンに搭載されたプロセッサー数を正しく出力してくれます。

逆に言うと、--all オプションを付けなければ正しい値を返さない場合があります。

```sh
$ taskset -c 0,1 nproc
2
```

なお `nproc` コマンドは GNU coreutils に含まれているので、Macでも Coreutils がインストールされていれば使用することができます。


### /proc/cpuinfo を使う (Linux)

/proc/cpuinfo の出力を grep します。

```sh
$ grep processor /proc/cpuinfo | wc -l
12
```


### sysctl コマンドを使う (Mac)

```sh
$ sysctl -n hw.logicalcpu_max
8
```
