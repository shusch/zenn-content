---
title: "GPGチートシート"
emoji: "🔭"
type: "tech"
topics: ["gpg", "openpgp", "cryptography", "linux"]
published: true
---

GPG (GnuPG) では鍵自体の作成や変更、鍵を用いた運用まで多くの使い方があります。

必要な場面で使うコマンドがいつもわからなくなるので、個人的に使う（使いそう）なコマンドをチートシートとしてまとめておきます。

## 主鍵と副鍵

GPGでは主鍵と副鍵に分けて管理することができる。

主鍵は副鍵への署名と失効をすることができ、公開鍵に署名を集めることで自分の信用情報となる (Web of Trust) 。
副鍵は万が一漏洩しても、主鍵を使って失効させることで、それまでの信用は保ったまま新しい副鍵を作成することができる。

副鍵に比べ主鍵の秘密鍵が漏洩した場合の被害は非常に大きいため、主鍵の役割は Certify にとどめてオフラインに退避させておく。万が一副鍵が漏洩した場合の失効や、アルゴリズムの危殆化などで鍵の交換（秘密鍵ごと交換）する際に後継鍵へ署名するのに使用する。

仮に主鍵が漏洩した場合でも、失効証明書をあらかじめ持っておけば失効させることは可能である。ただし、署名の検証や暗号化のために自分の公開鍵をインポートしていた全ての人に新たな公開鍵のインポートをしてもらう必要があるのと、鍵を作り直したことで Web of Trust の信用情報がゼロになってしまう。

そのため通常使用の副鍵と、副鍵の失効や Key transition 時の署名のみに使用する主鍵に分け、主鍵の秘密鍵はデバイスからは削除して、通常の運用を行う。

## 鍵の作成と主鍵の保管

### 主鍵の作成

`--generate-key` だとアルゴリズムや鍵用途の指定ができないので、`--full-generate-key` を使う。
（`--full-gen-key` は省略形）

`--expert` をしていないとアルゴリズムが古い場合があるので指定する。

```
$ gpg --full-gen-key --expert
gpg (GnuPG) 2.2.27; Copyright (C) 2021 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

# '(1) RSA and RSA (default)' などを指定すると証明用・署名用の鍵と暗号化用の鍵が同時に作成される
# 主鍵は証明用（副鍵に署名し、信頼できるものと証明する）に限定するため、 '(set your own capabilities)' なものを選択する
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

`--edit-key` からプロンプトに入り、作成する副鍵の分だけ `addkey` を実行する。

最後は `save` をして終了すること。saveしないと鍵が保存されない。

また副鍵を作成すると、主鍵の公開鍵も更新される。

```
$ gpg --expert --edit-key <keyid|fingerprint|email|name>
gpg (GnuPG) 2.2.27; Copyright (C) 2021 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Secret key is available.

sec  ed25519/12F618269D82E9B2
     created: 2024-05-01  expires: never       usage: C
     trust: ultimate      validity: ultimate
[ultimate] (1). Alice <alice@example.com>

# 署名用なら (10) 暗号化用なら (12) を選べば良い
# (11) を選んだ場合は、主鍵の例のようにトグルして用途を選択する
# (13) とすればインポート済みの鍵を使用することもできる
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

### 主鍵の秘密鍵を退避

主鍵が漏洩すると実質的に副鍵を含めたすべての情報が漏洩することになるため、オフラインの環境に退避させておく。

#### 主鍵の秘密鍵・失効証明書のバックアップ

1. 主鍵・副鍵の秘密鍵をエクスポートする

   ```
   gpg --export-secret-keys <KeyID> -a -o secret-all.key
   ```

1. 秘密鍵ファイルと失効証明書をバックアップする

1. エクスポートした秘密鍵ファイルはローカルから削除する

   ```
   rm secret-all.key
   # または
   shred -u secret-all.key
   ```

#### 主鍵の秘密鍵を削除

```
gpg --delete-secret-key <keyid|fingerprint|email|name>
```

削除後に `gpg -K` で秘密鍵の一覧を表示すると、主鍵 sec に `#` が付き使用できないことが確認できる。

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

```
cd ~/.gnupg/openpgp-revocs.d/
shred -u <fingerprint>.rev
```

## 複数のデバイスで鍵を共有する

### 秘密鍵のエクスポート

```
gpg --export-secret-keys <keyid|fingerprint|email|name> -a -o secret-all.key
```

この方法では主鍵・副鍵の両方の秘密鍵がエクスポートされる。
副鍵の秘密鍵のみをエクスポートしたい場合は、以下のコマンドを実行する。

```
gpg --export-secret-subkeys <keyid|fingerprint|email|name> -a -o secret-sub.key
```

### 公開鍵のエクスポート

インポートした公開鍵があり、それも共有したい場合は公開鍵をエクスポートする。
KeyID などを指定しない場合、すべての公開鍵をエクスポートする。

```
gpg --export -a -o public.key
```

### 鍵のインポート

```
gpg --import secret-all.key
gpg --import public.key
```

### エクスポートした鍵ファイルの削除

エクスポートした鍵ファイルは、忘れずにコピー元・コピー先の双方から削除する。

### 鍵の信用度を設定

自分で作成した鍵は究極的に信用できるため、 `ultimate (5)` で信用度を設定する。

## 鍵の失効

秘密鍵が漏洩した場合はただちに失効させる必要がある。
漏洩したのが主鍵の秘密鍵である場合は、失効させた後、完全に新しく主鍵から作り直すこと。

### 失効証明書の作成

鍵を作成したときに `~/.gnupg/openpgp-revocs.d/` に作成されている。
`--gen-revoke` で作った場合は失効理由も含めることができる。

```
gpg --gen-revoke <KeyID> -o <fingerprint>.rev
```

### 主鍵の失効

主鍵の秘密鍵が漏洩した場合は、用意しておいた失効証明書を使って直ちに秘密鍵ごと失効させる。

この方法では副鍵を含む鍵全体が失効する。

失効証明書を使用する際は、ファイルの以下部分の先頭のコロン (`:`) を削除して使用する。

```
:-----BEGIN PGP PUBLIC KEY BLOCK-----
```

インポートして失効させる。

```
gpg --import <revoke_cert_file>
```

#### 特定の副鍵のみ失効させる

```
gpg --edit-key <KeyID>

# 副鍵の上からの表示順か、KeyIDを指定する
# key 1 とすると1つ目の副鍵が指定される
gpg> key 1
gpg> revkey
gpg> save
```

副鍵の秘密鍵が漏洩した場合は、副鍵を削除しても意味はないため、失効させた上で新しい鍵を作成し直す。

#### 鍵が失効したことを公開する

鍵を失効させるだけではローカルな変更にとどまるので、失効したことを公開する必要がある。

失効させた鍵の公開鍵を鍵サーバに送信し、さらにGitHub等に登録しているGPGキーを置き換える。
これにより鍵の失効と新しい鍵の追加を周知することができる。

## 主鍵の交換

アルゴリズムの危殆化で鍵を変更する場合、主鍵の交換を行う。

新たな主鍵を作成した後は、正当に主鍵が移行したことを示すため、旧主鍵で新主鍵に対して署名を行う。
そして通常使用する副鍵を作成したら公開鍵を公開する。

新主鍵への署名後、旧主鍵は失効させる。
失効を行う際は、必要に応じて移行期間を空けて失効させる。

移行を表明するために、 Key Transition Statement を作成するのも良い。

## 自分の公開鍵を公開する

### 公開鍵をエクスポート

```
# 標準出力に表示
gpg --export -a <KeyID>

# ファイルに保存
gpg --export -a <KeyID> > public.asc
```

### 公開鍵を keyserver に公開

```
gpg --keyserver <KeyServer> --send-keys <KeyID>

# e.g.
gpg --keyserver keys.openpgp.org --send-keys <KeyID>
```

## 他人の公開鍵をインポートする

インポートする際には fingerprint が正しいものかを必ず検証する

### 鍵サーバからインポート

```
gpg --keyserver <KeyServer> --recv-keys <KeyID>

# e.g.
gpg --keyserver keys.openpgp.org --recv-keys <KeyID>
```

### Webページからインポート

```
gpg --fetch-keys https://github.com/octocat.gpg
```

### ファイルからインポート

```
gpg --import public.asc
```

### インポート前に指紋等の情報を確認する

公開鍵ファイルをインポートする前に指紋などの詳細情報を確認したい場合は、`pgpdump` や `gpgpdump` などのコマンドで確認できる。

## 鍵の操作

### 鍵の一覧と KeyID/Keygrip の確認

#### 秘密鍵の一覧

```
gpg -K
```

#### 公開鍵の一覧

```
gpg -k
```

#### 一覧に KeyID/Keygrip を表示する

KeyIDのフォーマットは `long` か `short` を指定できる。
fingerprint 末尾の文字列で、 long は16桁、short は8桁となる。

```
gpg -<k|K> --keyid-format <long|short>
```

Keygrip は `--with-keygrip` オプションで表示できる。

```
gpg -k --with-keygrip --keyid-format long
```

### 指紋の確認

```
gpg --fingerprint <keyid|fingerprint|email|name>
```

#### 副鍵の指紋を確認

`fingerprint` を2つ入力すると、副鍵の指紋も表示される

```
gpg --fingerprint --fingerprint <keyid|fingerprint|email|name>
```

### 公開鍵への署名を確認

ある公開鍵に対しての署名を確認する

```
gpg --list-sigs <keyid|fingerprint|email|name>
```

公開鍵の一覧に署名も表示する

```
gpg --check-sigs
```

### 鍵の信用度を変更する

編集する公開鍵を指定して、GPGのプロンプトに入る。

```
gpg --edit-key <keyid|fingerprint|email|name>
```

信用度を設定する（信用度は適切なものを選択する）

```
gpg> trust
pub  ed25519/ABCDEFGHIJKLMNOP
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

最後に save して保存して終了する。
信用度の変更だけであれば quit でもOK。

### 他人の公開鍵への署名

#### ローカルな署名

エクスポート不可なもの（署名したことを公開しない）として署名が行われる。
特にエクスポートする必要性がなければ、この方法で署名する。

```
gpg --lsign-key <KeyID>
```

#### 一般的な署名

外部公開可能なものとして署名する。
この場合、自分がこの鍵を信用したとして公開されるため、公開鍵の指紋を必ず確認し、かつ署名が必要な状態において行う。

```
gpg --sign-key <KeyID>

# 署名に使う秘密鍵を指定する場合
gpg -u <KeyID> --sign-key <KeyID>
```

### 鍵の削除

#### 秘密鍵を削除する

```
gpg --delete-secret-keys <KeyID>

# 削除されたか確認
gpg -K
```

#### 公開鍵を削除する

```
gpg --delete-keys <KeyID>

# 削除されたか確認
gpg -k
```

### 有効期限の変更

有効期限を変更すると公開鍵も変わるため、必要に応じて公開している鍵も更新する。

```
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

### パスフレーズの変更

パスフレーズは後からの変更や、端末ごとに異なるパスフレーズにすることができる。

```
gpg --passwd <keyid|fingerprint|email|name>
```

## ファイルの署名と検証

署名を検証する際、署名した秘密鍵に対応する公開鍵を鍵サーバかファイルからインポートしておく。

署名の検証時、以下のように `[unknown]` や WARNING が出る場合がある。

> ```
> gpg: Signature made Mon 02 Jan 2006 03:04:05 PM MST
> gpg:                using RSA key 1234567890ABCDEF
> gpg: Good signature from "Alice <alice@example.com>" [unknown]
> gpg: WARNING: This key is not certified with a trusted signature!
> gpg:          There is no indication that the signature belongs to the owner.
> Primary key fingerprint: ABCD EFGH IJKL MNOP QRST  UVWX YZ01 2345 6789 ABCD
> ```

これは署名の検証には成功したが、公開鍵自体の信用度が不明なため。
公開鍵に対して信用度を設定するか、署名を行うことで回避可能。

### 署名

署名付きファイルの作成

```
# バイナリ形式
gpg -s file.txt
gpg --sign file.txt

# アスキー形式
gpg -sa file.txt
gpg --sign --armor file.txt
```

本文は平文のテキストで、署名を付加したファイルの作成

```
gpg --clearsign file.txt
```

元ファイルと署名ファイルを分離

```
gpg -b file.txt
gpg --detach-sign file.txt

# アスキー形式
gpg -ba file.txt
gpg --detach-sign --armor file.txt
```

署名用の秘密鍵が複数ある場合

```
gpg -u <秘密鍵の情報> -sa <署名対象ファイル>
```

### 検証

署名ファイルの検証

```
gpg --verify file.txt.gpg
```

分離署名の検証

```
gpg --verify <署名ファイル> <データファイル>
```

## ファイルの暗号化と復号化

暗号ファイルを共有する相手の公開鍵を用いて暗号化するため、事前に公開鍵をインポートしておく必要がある。

### 暗号化

-r オプションで、暗号ファイルを受け取る人のインポート済み公開鍵の情報 (KeyID, fingerprint, Email, name) を入力する。

```
gpg -e -r <公開鍵の情報> <暗号対象ファイル>

# 受信者が複数いる場合
gpg -e -r <公開鍵の情報1> -r <公開鍵の情報2> <暗号対象ファイル>
```

### 復号化

```
gpg <暗号化ファイル>
gpg -d <暗号化ファイル>
gpg --decrypt <暗号化ファイル>
```

## ファイルの署名と暗号化を同時に行う

暗号化のみだとなりすましの可能性があるため、暗号化と同時に署名も行うことが理想的である。

```
gpg -se -r <受信者の公開鍵情報> <暗号対象ファイル>
```

署名検証と復号化は特にオプションを指定しなくて良い。

```
gpg <対象ファイル>
```

## Git での署名

### コミットへの署名

```
git commit -S
```

マージコミットへの署名

```
git merge -S <branch>

# 有効な署名のみマージする場合
git merge --verify-signatures -S <branch>
```

### コミットの署名を検証

```
git log --show-signature
```

### タグへの署名

`-a` オプションの代わりに `-s` を使用するとタグに署名できる。

```
git tag -s v1.2.3 -m 'tag annotation'
```

`git show` で署名を表示することができる。

```
git show <tag>
```

### タグの署名を検証

署名者の公開鍵がインポート済みである必要がある。

```
git tag -v <tag>
```
