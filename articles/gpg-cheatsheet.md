---
title: "GPGチートシート"
emoji: "🔭"
type: "tech"
topics: ["gpg", "openpgp", "cryptography", "linux"]
published: true
---

GPG (GnuPG) では鍵自体の作成や変更、鍵を用いた運用まで多くのコマンドがあります。

必要な場面で使うコマンドがいつもわからなくなるので、個人的に使う（使いそう）なコマンドをチートシートとしてまとめておきます。


## 鍵の種類

GPGで作成できる鍵の種類には、以下の4種類があります。

- Certify（証明）
- Sign（署名）
- Authenticate（認証）
- Encrypt（暗号化）

それぞれの役割は以下の通りです。

### Certify（証明）

公開鍵へ署名をするために使います。

署名をすることで、自分がその鍵に対して信用していることを証明します。
具体的には、自分の副鍵または他人の公開鍵に対して署名を行います。

ほかには自分の鍵を失効させる場合も、無効化したことを署名するために使います。

### Sign（署名）

ファイルなどのデータに対して電子署名を行うために使います。

電子署名を行うことで、鍵の所有者本人であることの検証や改ざん検知を可能にします。

### Authenticate（認証）

SSH などのユーザー認証に使います。

### Encrypt（暗号化）

データの暗号化のために使います。


## 主鍵と副鍵

GPGで使用する鍵は**主鍵**と**副鍵**の2つがあります。

それぞれの鍵には異なる役割を与えることができ、適切に使い分けることで安全性を高めることができます。
一つ一つの主鍵・副鍵ごとに秘密鍵と公開鍵がそれぞれ存在します。

### 主鍵

主鍵は最低限 `Certify`（証明）の機能を持ち、ほかに `Sign`（署名）、 `Authenticate`（認証）の機能も付与することができます。
主鍵の公開鍵に他者からの署名を集めることで、それが自分の信用情報となります (Web of Trust) 。

副鍵に比べ主鍵の秘密鍵が漏洩した場合の被害は非常に大きいため、主鍵の役割は `Certify` にとどめてオフラインに退避させておくと安全性が高まります。

仮に主鍵が漏洩した場合でも、失効証明書をあらかじめ持っておけば失効させることは可能です。
ただし、署名の検証や暗号化のために自分の公開鍵をインポートしていた全ての人に新たな公開鍵のインポートをしてもらう必要があるのと、鍵を作り直したことで Web of Trust の信用情報がゼロになってしまいます。可能な限り主鍵は交換せずに使い続けられることが望ましいといえます。

そのため通常使用の副鍵と、副鍵の失効や Key transition 時の署名のみに使用する主鍵に分け、主鍵の秘密鍵はデバイスからは削除して、通常の運用を行うことが望ましいでしょう。

### 副鍵

副鍵は、主鍵と異なり `Encrypt`（暗号化）の機能を持たせることができます。ほかにも `Sign`（署名）、 `Authenticate`（認証）の機能を持った鍵も作成することが可能です。

通常GPGで使用する署名や暗号化などの機能は副鍵側に持たせ、さらに機能ごとに鍵自体も分けておくことが多いです。
これにより鍵が漏洩した場合のリスクを抑えることができます。

また副鍵は万が一漏洩しても、主鍵を使って失効させることで、それまでに得た Web of Trust の信用は保ったまま新しい副鍵を作成することができます。


## 鍵の作成と主鍵の保管

### 主鍵の作成

主鍵の作成には `--generate-key` オプションが存在します。
しかしこれだとアルゴリズムや鍵用途の指定ができないので、`--full-generate-key` を使いましょう。
（`--full-gen-key` は省略形）

また `--expert` を付けないとアルゴリズムが古い場合があるので指定します。

主鍵を作成する際は、パスフレーズの入力を求められます。
SSH鍵などではパスフレーズを省略してしまうことも多いですが、GPG 鍵では必ず設定しましょう。主鍵が流出した場合、最後に秘密鍵を守ってくれるものになります。

```sh
$ gpg --full-gen-key --expert
gpg (GnuPG) 2.2.27; Copyright (C) 2021 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

# '(1) RSA and RSA (default)' などを指定すると
# 証明用・署名用の主鍵と暗号化用の副鍵が同時に作成されます
#
# 主鍵は証明用（副鍵に署名し、信頼できるものと証明する）に限定したいので、
# '(set your own capabilities)' なものを選択します
#
# 同じ暗号強度でも鍵長が短い楕円曲線暗号を使うのがおすすめです
# その場合 (11) ECC を指定します
gpg: directory '/home/user/.gnupg' created
gpg: keybox '/home/user/.gnupg/pubring.kbx' created
Please select what kind of key you want:
   (1) RSA and RSA (default)
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
   (9) ECC and ECC
  (10) ECC (sign only)
  (11) ECC (set your own capabilities)
  (13) Existing key
  (14) Existing key from card
Your selection? 11

Possible actions for a ECDSA/EdDSA key: Sign Certify Authenticate
Current allowed actions: Sign Certify

   (S) Toggle the sign capability
   (A) Toggle the authenticate capability
   (Q) Finished

# 鍵への署名だけできれば良く Sign はいらないのでオフ
Your selection? S

Possible actions for a ECDSA/EdDSA key: Sign Certify Authenticate
Current allowed actions: Certify

   (S) Toggle the sign capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? Q

# 鍵のアルゴリズムを指定します
# 現代の標準は Curve25519 を使用した Ed25519 なので (1) を指定すれば良いでしょう
Please select which elliptic curve you want:
   (1) Curve 25519
   (3) NIST P-256
   (4) NIST P-384
   (5) NIST P-521
   (6) Brainpool P-256
   (7) Brainpool P-384
   (8) Brainpool P-512
   (9) secp256k1
Your selection? 1
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 0
Key does not expire at all
Is this correct? (y/N) y

GnuPG needs to construct a user ID to identify your key.

# GPG では自身を証明するものになるので、本名フルネームを指定しましょう
Real name: Alice
Email address: alice@example.com
Comment:
You selected this USER-ID:
    "Alice <alice@example.com>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? O
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.
gpg: /home/user/.gnupg/trustdb.gpg: trustdb created
gpg: key 12F618269D82E9B2 marked as ultimately trusted
gpg: directory '/home/user/.gnupg/openpgp-revocs.d' created
gpg: revocation certificate stored as '/home/user/.gnupg/openpgp-revocs.d/6CAFCE241886D944F703A10212F618269D82E9B2.rev'
public and secret key created and signed.

pub   ed25519 2024-05-01 [C]
      6CAFCE241886D944F703A10212F618269D82E9B2
uid                      Alice <alice@example.com>
```

### 副鍵の作成

`--edit-key` からプロンプトに入り、作成する副鍵の分だけ `addkey` を実行します。

最後は `save` をしてから終了しましょう。saveしないと鍵が保存されません。

また副鍵を作成すると、主鍵の公開鍵も更新されます。

```sh
$ gpg --edit-key --expert <keyid|fingerprint|email|name>
gpg (GnuPG) 2.2.27; Copyright (C) 2021 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Secret key is available.

sec  ed25519/12F618269D82E9B2
     created: 2024-05-01  expires: never       usage: C
     trust: ultimate      validity: ultimate
[ultimate] (1). Alice <alice@example.com>

# 署名用なら (10) 暗号化用なら (12) を選べば良いです
# (11) を選んだ場合は、主鍵の例のようにトグルして用途を選択します
# (13) とすればインポート済みの鍵を使用することもできます
gpg> addkey
Please select what kind of key you want:
   (3) DSA (sign only)
   (4) RSA (sign only)
   (5) Elgamal (encrypt only)
   (6) RSA (encrypt only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
  (10) ECC (sign only)
  (11) ECC (set your own capabilities)
  (12) ECC (encrypt only)
  (13) Existing key
  (14) Existing key from card
Your selection? 10

# 主鍵と同様に楕円曲線暗号 ECC を選択します
Please select which elliptic curve you want:
   (1) Curve 25519
   (3) NIST P-256
   (4) NIST P-384
   (5) NIST P-521
   (6) Brainpool P-256
   (7) Brainpool P-384
   (8) Brainpool P-512
   (9) secp256k1
Your selection? 1
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 0
Key does not expire at all
Is this correct? (y/N) y
Really create? (y/N) y
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.

sec  ed25519/12F618269D82E9B2
     created: 2024-05-01  expires: never       usage: C
     trust: ultimate      validity: ultimate
ssb  ed25519/ABF7A3BE4E8ABB5B
     created: 2024-05-01  expires: never       usage: S
[ultimate] (1). Alice <alice@example.com>

gpg> save
```

これで証明用の主鍵と署名用の副鍵ができました。
暗号化と認証用の副鍵を作る場合も、同様にしてつくればOKです。

### 主鍵の秘密鍵を退避

主鍵が漏洩すると実質的に副鍵を含めたすべての情報が漏洩することになるため、オフラインの環境に退避させておく方法を紹介します。

#### 主鍵の秘密鍵・失効証明書のバックアップ

1. 主鍵・副鍵の秘密鍵をエクスポートする

   ```sh
   gpg --export-secret-keys -a -o secret-all.key <KeyID>
   ```

1. 秘密鍵ファイルと失効証明書をバックアップする

1. エクスポートした秘密鍵ファイルはローカルから削除する

   ```sh
   rm secret-all.key
   # または
   shred -u secret-all.key
   ```

#### 主鍵の秘密鍵を削除

```sh
gpg --delete-secret-key <keyid|fingerprint|email|name>
```

削除後に `gpg -K` で秘密鍵の一覧を表示すると、主鍵 sec に `#` が付き使用できないことが確認できます。

```
$ gpg -K
/home/user/.gnupg/pubring.kbx
----------------------------
sec#  ed25519 2012-01-23 [C]
      6CAFCE241886D944F703A10212F618269D82E9B2
uid           [  究極  ] Alice <alice@example.com>
ssb   cv25519 2012-01-23 [E]
ssb   ed25519 2012-01-23 [S]
```

#### 失効証明書の削除

```sh
cd ~/.gnupg/openpgp-revocs.d/
shred -u <fingerprint>.rev
```


## 複数のデバイスで鍵を共有する

### 秘密鍵のエクスポート

以下のコマンドで副鍵の秘密鍵をエクスポートします（主鍵の秘密鍵はエクスポートされません）。

```sh
gpg --export-secret-subkeys -a -o secret-sub.key <keyid|fingerprint|email|name>
```

もし主鍵・副鍵の両方の秘密鍵をエクスポートしたい場合は、以下のコマンドを実行します。

```sh
gpg --export-secret-keys -a -o secret-all.keys <keyid|fingerprint|email|name>
```

### 公開鍵のエクスポート

自分の鍵をエクスポートする場合は秘密鍵をエクスポートすれば良いですが、他者の公開鍵をインポートしていた場合にそれも共有したい場合は公開鍵をエクスポートしましょう。

KeyID などを指定しない場合、すべての公開鍵をエクスポートします。

```sh
# すべての公開鍵をエクスポート
gpg --export -a -o public.key

# 指定した公開鍵のみエクスポート
gpg --export -a -o public.key <keyid|fingerprint|email|name>
```

### 鍵のインポート

```sh
gpg --import secret-all.key
gpg --import public.key
```

### エクスポートした鍵ファイルの削除

エクスポートした鍵ファイルは、忘れずにコピー元・コピー先の双方から削除しましょう。

### 鍵の信用度を設定

自分で作成した鍵は究極的に信用できるため、 `ultimate (5)` で信用度を設定します。

[信用度の設定方法](#鍵の信用度を変更する)


## 鍵の失効

秘密鍵が漏洩した場合はただちに失効させる必要があります。
漏洩したのが主鍵の秘密鍵である場合は、鍵を失効させた後、完全に新しく主鍵から作り直しましょう。

### 失効証明書の作成

鍵を作成したときに `~/.gnupg/openpgp-revocs.d/` に作成されています。
`--gen-revoke` で作った場合は失効理由も含めることができます。

```sh
gpg --gen-revoke -o <fingerprint>.rev <KeyID>
```

### 主鍵の失効

主鍵の秘密鍵が漏洩した場合は、用意しておいた失効証明書を使って直ちに秘密鍵ごと失効させましょう。

この方法では副鍵を含む鍵全体が失効します。

失効証明書を使用する際は、ファイルの以下部分の先頭のコロン (`:`) を削除して使用します。

```
:-----BEGIN PGP PUBLIC KEY BLOCK-----
```

インポートすることで鍵が失効します。

```sh
gpg --import <revoke_cert_file>
```

#### 特定の副鍵のみ失効させる

```sh
gpg --edit-key <KeyID>

# 副鍵の上からの表示順か、KeyIDを指定します
# key 1 とすると1つ目の副鍵が指定されます
gpg> key 1
gpg> revkey
gpg> save
```

副鍵の秘密鍵が漏洩した場合は、副鍵を削除しても意味はないため、失効させた上で新しい鍵を作成し直しましょう。

#### 鍵が失効したことを公開する

鍵を失効させるだけではローカルな変更にとどまるので、失効したことを公開する必要があります。

失効させた鍵の公開鍵を鍵サーバに送信し、さらにGitHub等に登録しているGPGキーを置き換えます。
これにより鍵の失効と新しい鍵の追加を周知することができます。


## 主鍵の交換

アルゴリズムの危殆化で鍵を変更する場合、主鍵の交換を行います。

新たな主鍵を作成した後は、正当に主鍵が移行したことを示すため、旧主鍵で新主鍵に対して署名を行いましょう。
そして通常使用する副鍵を作成したら公開鍵を公開します。

新主鍵への署名後、旧主鍵は失効させます。
失効を行う際は、必要に応じて移行期間を空けて失効させます。

移行を表明するために、 Key Transition Statement を作成するのも良いでしょう。


## 自分の公開鍵を公開する

### 公開鍵をエクスポート

```sh
# 標準出力に表示
gpg --export -a <KeyID>

# ファイルに保存
gpg --export -a -o public.asc <KeyID>
```

### 公開鍵を keyserver に公開

```sh
gpg --keyserver <KeyServer> --send-keys <KeyID>

# e.g.
gpg --keyserver keys.openpgp.org --send-keys <KeyID>
```


## 鍵の一覧の確認

### 秘密鍵の一覧

```sh
gpg -K
```

### 公開鍵の一覧

```sh
gpg -k
```

### 一覧に KeyID/Keygrip を表示する

KeyIDのフォーマットは `long` か `short` を指定できます。
これは fingerprint 末尾の文字列で、 long だと16桁、short だと8桁で表示されます。

```sh
gpg -<k|K> --keyid-format <long|short>
```

Keygrip は `--with-keygrip` オプションで表示できます。

```sh
gpg -k --with-keygrip --keyid-format long
```


## フィンガープリントの確認

```sh
gpg --fingerprint <keyid|fingerprint|email|name>
```

### 副鍵の指紋を確認

`fingerprint` を2つ入力すると、副鍵の指紋も表示されます。

```sh
gpg --fingerprint --fingerprint <keyid|fingerprint|email|name>
```


## 他人の公開鍵をインポートする

インポートする際には fingerprint が正しいものかを必ず検証しましょう。

### 鍵サーバからインポート

```sh
gpg --keyserver <KeyServer> --recv-keys <KeyID>

# e.g.
gpg --keyserver keys.openpgp.org --recv-keys <KeyID>
```

### Webページからインポート

```sh
gpg --fetch-keys https://github.com/web-flow.gpg
```

### ファイルからインポート

```sh
gpg --import public.asc
```

### インポート前に指紋等の情報を確認する

公開鍵ファイルをインポートする前に指紋などの詳細情報を確認したい場合は、`pgpdump` や `gpgpdump` などのコマンドで確認できます。


## 鍵の信用度を変更する

編集する公開鍵を指定して、GPGのプロンプトに入ります。

```sh
gpg --edit-key <keyid|fingerprint|email|name>
```

信用度を設定します（信用度は適切なものを選択する）。

```sh
gpg> trust
pub  ed25519/ABCDEF1234567890
     作成: 2012-01-23  有効期限: 無期限       利用法: SC
     信用: 不明の     有効性: 不明の
sub  cv25519/1234567890ABCDEF
     作成: 2012-01-23  有効期限: 無期限       利用法: E
[  不明  ] (1). Alice <alice@example.com>

他のユーザの鍵を正しく検証するために、このユーザの信用度を決めてください
(パスポートを見せてもらったり、他から得たフィンガープリントを検査したり、などなど)

  1 = 知らない、または何とも言えない
  2 = 信用し ない
  3 = まぁまぁ信用する
  4 = 充分に信用する
  5 = 究極的に信用する
  m = メーン・メニューに戻る

あなたの決定は? 5
本当にこの鍵を究極的に信用しますか? (y/N) y

gpg>
```

最後に save して保存して終了します。
信用度の変更だけであれば quit でもOK。

### trust（信用度）と validity（有効性）の違い

trust は自分がその公開鍵をどの程度信用できるかを示す値です。
自分自身の公開鍵は 5 とし、他人の公開鍵は 1 から 4 のいずれかを指定します。

validity は trust の値や署名をもとに、その鍵が正しいものか、安全に通信できるかをGPGが自動で判断します。

validity が低い公開鍵は、GPGが信用できないと判断し、署名の検証や暗号化の際に警告が表示されることがあります。


## 公開鍵への署名を確認

ある公開鍵に対しての署名を確認する

```sh
gpg --list-sigs <keyid|fingerprint|email|name>
```

公開鍵の一覧に署名も表示する

```sh
gpg --check-sigs
```


## 他人の公開鍵への署名

公開鍵への署名には `Certify` 機能を持つ秘密鍵が必要になります。
`Sign` 機能の副鍵では署名できないので注意しましょう。

### ローカルな署名

エクスポート不可なもの（署名したことを公開しない）として署名が行われます。
特にエクスポートする必要性がなければ、この方法で署名しましょう。

```sh
gpg --lsign-key <KeyID>
```

### 一般的な署名

外部公開可能なものとして署名します。
この場合、自分がこの鍵を信用したとして公開されるため、公開鍵の指紋を必ず確認し、かつ署名が必要な状態において行います。

```sh
gpg --sign-key <KeyID>

# 署名に使う秘密鍵を指定する場合
gpg -u <KeyID> --sign-key <KeyID>
```


## 鍵の削除

### 秘密鍵を削除する

```sh
gpg --delete-secret-keys <KeyID>

# 削除されたか確認
gpg -K
```

### 公開鍵を削除する

```sh
gpg --delete-keys <KeyID>

# 削除されたか確認
gpg -k
```


## 有効期限の変更

有効期限を変更すると公開鍵も変わるため、必要に応じて公開している鍵も更新します。

```sh
gpg --edit-key <keyid|fingerprint|email|name>

# 副鍵の上からの表示順か、KeyIDを指定する
# key 1 とすると1つ目の副鍵が指定される
gpg> key 1

gpg> expire

Changing expiration time for a subkey.
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) # 有効期限を入力する e.g. 1y

# パスフレーズを入力して、保存する
gpg> save
```


## パスフレーズの変更

パスフレーズは後からの変更や、端末ごとに異なるパスフレーズにすることができます。

```sh
gpg --passwd <keyid|fingerprint|email|name>
```


## ファイルの署名と検証

署名を検証する際、署名した秘密鍵に対応する公開鍵を鍵サーバかファイルからインポートしておきます。

署名の検証時、以下のように `[unknown]` や WARNING が出る場合があります。

> ```sh
> gpg: Signature made Mon 02 Jan 2006 03:04:05 PM MST
> gpg:                using RSA key 1234567890ABCDEF
> gpg: Good signature from "Alice <alice@example.com>" [unknown]
> gpg: WARNING: This key is not certified with a trusted signature!
> gpg:          There is no indication that the signature belongs to the owner.
> Primary key fingerprint: ABCD EF12 3456 7890 ABCD  EF12 3456 7890 ABCD EF01
> ```

これは署名の検証には成功したが、公開鍵自体の信用度が不明なためです。
公開鍵に対して信用度を設定するか、署名を行うことで回避可能となります。

### 署名

署名付きファイルの作成

```sh
# バイナリ形式
gpg -s file.txt
gpg --sign file.txt

# アスキー形式
gpg -sa file.txt
gpg --sign --armor file.txt
```

本文は平文のテキストで、署名を付加したファイルの作成

```sh
gpg --clearsign file.txt
```

元ファイルと署名ファイルを分離

```sh
gpg -b file.txt
gpg --detach-sign file.txt

# アスキー形式
gpg -ba file.txt
gpg --detach-sign --armor file.txt
```

署名用の秘密鍵が複数ある場合

```sh
gpg -u <秘密鍵の情報> -sa <署名対象ファイル>
```

### 検証

署名ファイルの検証

```sh
gpg --verify file.txt.gpg
```

分離署名の検証

```sh
gpg --verify <署名ファイル> <データファイル>
```


## ファイルの暗号化と復号化

暗号化には暗号ファイルを共有する相手の公開鍵を使用するため、事前に公開鍵をインポートしておく必要があります。

なおGPGでの暗号化そのものは、セッション鍵を使った共通鍵暗号方式にて行います。
暗号化用の副鍵は、セッション鍵の鍵交換のために使用されています。

### 暗号化

-r オプションで、暗号ファイルを受け取る人のインポート済み公開鍵の情報 (KeyID, fingerprint, Email, name) を入力します。

```sh
gpg -e -r <公開鍵の情報> <暗号対象ファイル>

# 受信者が複数いる場合
gpg -e -r <公開鍵の情報1> -r <公開鍵の情報2> <暗号対象ファイル>
```

### 復号化

```sh
gpg <暗号化ファイル>
gpg -d <暗号化ファイル>
gpg --decrypt <暗号化ファイル>
```


## ファイルの署名と暗号化を同時に行う

暗号化のみだとなりすましの可能性があるため、暗号化と同時に署名も行うのが良いでしょう。

```sh
gpg -se -r <受信者の公開鍵情報> <暗号対象ファイル>
```

署名検証と復号化は特にオプションを指定しなくて良いです。

```sh
gpg <対象ファイル>
```


## Git での署名

### コミットへの署名

```sh
git commit -S
```

マージコミットへの署名

```sh
git merge -S <branch>

# 有効な署名のみマージする場合
git merge --verify-signatures -S <branch>
```

### コミットの署名を検証

```sh
git log --show-signature
```

### タグへの署名

`-a` オプションの代わりに `-s` を使用するとタグに署名できます。

```sh
git tag -s v1.2.3 -m 'tag annotation'
```

`git show` で署名を表示できます。

```sh
git show <tag>
```

### タグの署名を検証

署名者の公開鍵がインポート済みの必要があります。

```sh
git tag -v <tag>
```
