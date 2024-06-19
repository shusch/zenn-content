---
title: "CloudflareのwwwリダイレクトルールをTerraformで作成する"
emoji: "🍤"
type: "tech"
topics: ["cloudflare", "cloudflarepages", "terraform"]
published: true
---

## はじめに

`example.com` から `www.example.com` へのリダイレクト（wwwなしへのリダイレクトでも同じ）は、Cloudflare DNS で管理されたサイトであれば、ブラウザコンソールからリダイレクトルールを作成することで簡単に実現することができます。

ただこれをTerraformで実現する方法は、情報があまり無かったのでまとめておきます。

## 動作環境

- Terraform v1.8.4
- Cloudflare Provider v4.34.0

## Cloudflare × Terraform のツラミ

いきなりですが、少なくとも現時点では Cloudflare のリソースをTerraformできれいに構築する、というのは非常に難易度が高いです。

Cloudflare自体の開発が活発であり、機能そのものやサービス名が変わることがあります。そのため世の中の情報が古くなっていることが多いです。

また Cloudflare のドキュメントと Terraform Provider のドキュメントをそれぞれ照らし合わせても、微妙に名称が違っていたり、機能ごとの関係性がマネジメントコンソール上とTerraformのリソース定義とで異なっているため推測も非常に困難だったりします。

そのためドキュメントを漁ったうえで、結局はマネジメントコンソール上から作成し、それを `terraform import` でインポートしてからリソース定義を書く。という手順を踏むことを覚悟しておいたほうがよいです。

今回のリダイレクトルールも試行錯誤の上、結局はコンソールで作成したものをインポートするというハメになりました。

## 結論

結論から書くと、以下のようなリソース定義を記述すればリダイレクトルールが作成できます。

ドメイン(`example.com`) が書かれている2箇所は、自分が持っているドメインに置き換えてください。

作成されたリソースは、ブラウザコンソールからだと
Webサイト > （左メニューの）ルール > リダイレクトルール
から見ることができます。

```hcl:main.tf
resource "cloudflare_ruleset" "redirect" {
  kind    = "zone"
  name    = "default"
  phase   = "http_request_dynamic_redirect"
  zone_id = var.cloudflare_zone_id

  rules {
    action = "redirect"
    action_parameters {
      from_value {
        preserve_query_string = true
        status_code           = 301
        target_url {
          expression = "concat(\"https://www.example.com\", http.request.uri.path)"
        }
      }
    }
    description = "redirect non-www to www"
    enabled     = true
    expression  = "(http.host eq \"example.com\")"
  }
}
```

```hcl:variables.tf
# 実際の値は `terraform.tfvars` などから取得する想定で記述しています
# 運用に合わせて上述の zone_id は定義してください
variable "cloudflare_zone_id" {
  type = string
}
```

### 前提条件

- Cloudflare DNS を使用していること
- リダイレクト元と先のドメインともに Aレコード or CNAMEレコードを作成済みで、Cloudflare DNS のプロキシをオンにしていること
- リダイレクト先のドメイン（ここでは `www.example.com`）ですでにアクセスできること
  - リダイレクト元のドメイン（ここでは `example.com`）は、www ドメインと同じレコードにしても良いですし、どのみちリダイレクトしてしまうので `192.0.2.1` などのダミーIPをAレコードに設定しても問題ないと思われます

また必須条件ではないと思いますが、サイトは Cloudflare Pages で構築している想定です。

### 補足説明

基本的に上記のコードのドメインだけ変えれば動くのですが、いくつか補足をします。

詳細は [公式のTerraformドキュメント](https://registry.terraform.io/providers/cloudflare/cloudflare/latest/docs/resources/ruleset) を参照してください。
※最新バージョンへのリンクなので、一部本記事のコードとは異なる可能性があります

1. `name` の定義

   分かりづらいのですが、ここでの name はブラウザコンソールの画面には登場しません！
   これはルールセットに対する名称で、現状では子階層の rule しか画面には出てこないので特に意識しなくても良いと思います。

   ブラウザから作成した場合、"default" が入っていたのでこれをそのまま使っています。
   （ブラウザから作成したリソース定義を確認する方法は後述）

1. `rules.description` の定義

   `rules` 配下の `description` の定義は、一見画面上に出てこない補足的な情報かと思いきや、実際にブラウザコンソール上に表示されるのはこの値です(!!)

   区別しやすいような名称は、上述の name よりむしろこちらに付けるべきでしょう。

## ここまでのまとめ

リソースの定義方法自体は、ここまでの内容を見ればおしまいです。

ここから先では、この定義方法に行き着くまでにやったことを書いていきます。
なのでさくっとリダイレクトルールだけ作りたい方はここまでの内容を見れば十分です。

とはいえ、今後Terraformで別のCloudflareサービスを構築したい場合に、同じように情報が得られず、どのようにしてリソース定義を書けばいいか躓いた時には参考になるかと思います。

## Cloudflareのリダイレクトルール

さて、Cloudflareでルートドメインからwwwドメインへのリダイレクト（もしくはその逆）というと、Page Rules を使った情報が執筆時点（2024年6月）ではほとんどでしょう。

しかしながら、Page Rules はすでに Legacy 扱いで、公式からも代替のルールを使用することが推奨されており、移行ガイドまで作られています。

https://developers.cloudflare.com/rules/page-rules/

そのため Page Rules の代替となるルールを探していくことになります。

[移行ガイド](https://developers.cloudflare.com/rules/reference/page-rules-migration/) からドキュメントを漁ると、Terraformでリダイレクトをしている例がありました。

https://developers.cloudflare.com/rules/url-forwarding/single-redirects/terraform-example/

どうやら `cloudflare_ruleset` というリソースを使用すれば良いようです。
しかし、このドキュメントの例だと単一のページをリダイレクトさせる例なので、もう少し調査が必要そうです。

## `cloudflare_ruleset` のリソース定義

`cloudflare_ruleset` のドキュメントを見ても、non-www to www ドメインへのリダイレクトの例はありませんでした。

https://registry.terraform.io/providers/cloudflare/cloudflare/latest/docs/resources/ruleset

以下にダッシュボードから手動で作成する場合に、リダイレクトルールを作成する方法を見つけました。
（例は wwwドメイン→ルートドメイン へのリダイレクトですが大きくは変わりません）

これとTerraformのドキュメントを照らし合わせるのですが、ところどころで用語の使い方が違っているのでどのTerraformスキーマがダッシュボード側と対応しているのかが掴みきれませんでした。

https://developers.cloudflare.com/rules/reference/page-rules-migration/#migrate-forwarding-url

ここでTerraformで構築するのは諦め、ダッシュボードから作成したものをインポートする方針に切り替えることになります…

## ダッシュボードからリダイレクトルールを作成

これは特段の説明はありません。

先程のドキュメント通りにルールを作成していきます。
ドキュメントだとwwwドメインからルートドメインへのリダイレクトになっていることにだけ注意が必要です。
今回の場合、逆方向へのリダイレクトを行いたかったので、適当に読み替えてルールを作成しました。

https://developers.cloudflare.com/rules/reference/page-rules-migration/#migrate-forwarding-url


## terraform import でリソース定義をインポート

リダイレクトルールは作成できたので、これを `terraform import` コマンドを使ってTerraformのコードに持ってきます。

import コマンドはドキュメントにあるのですが、`account scoped Ruleset` と `zone scoped Ruleset` があるようで、更には `ruleset_id` も必要らしくここでも躓きます…

```sh:terraform import コマンド
# Import an account scoped Ruleset configuration.
$ terraform import cloudflare_ruleset.example account/<account_id>/<ruleset_id>

# Import a zone scoped Ruleset configuration.
$ terraform import cloudflare_ruleset.example zone/<zone_id>/<ruleset_id>
```

https://registry.terraform.io/providers/cloudflare/cloudflare/latest/docs/resources/ruleset#import

リダイレクトルールの編集画面URLにあるIDらしきものでimportしてみるも、これは失敗。

```sh
$ terraform import cloudflare_ruleset.www_redirects_main zone/<zone_id>/<ruleset_id らしきもの>
╷
│ Error: Cannot import non-existent remote object
│
│ While attempting to import an existing object to "cloudflare_ruleset.redirects", the
│ provider detected that no object exists with the given id. Only pre-existing objects can be
│ imported; check that the id is correct and that it is associated with the provider's configured
│ region or endpoint, or use "terraform apply" to create a new remote object for this resource.
╵
```

### APIドキュメントから rulesets のIDを特定する

APIドキュメントで "list rulesets" などのキーワードで検索すると、それらしきドキュメントを発見しました。

https://developers.cloudflare.com/api/operations/listZoneRulesets

ドキュメントを元にリクエストを投げてみます。
※AuthorizationヘッダにはAPIトークンを指定します
※APIトークンには、[ドキュメント](https://developers.cloudflare.com/fundamentals/api/reference/permissions/) を参考に権限を付与しておく必要があります

以下がレスポンス例です（実際にはもう少しresultが返ってきましたが長いので一部省略）。

```json
{
  "result": [
    {
      "id": "********************************",
      "name": "Cloudflare Normalization Ruleset",
      "description": "Created by the Cloudflare security team, this ruleset provides normalization on the URL path",
      "kind": "managed",
      "version": "X",
      "last_updated": "2023-05-02T16:19:06.119072Z",
      "phase": "http_request_sanitize"
    },
    {
      "id": "********************************",
      "name": "default",
      "description": "",
      "kind": "zone",
      "version": "1",
      "last_updated": "2024-06-11T15:05:06.408172Z",
      "phase": "http_request_dynamic_redirect"
    },
  ],
  "success": true,
  "errors": [],
  "messages": []
}
```

明らかに自分で作成していないリソースがずらずらと返ってきたので、間違えたか…と思いつつ、`last_updated` がリダイレクトルールの編集時刻と思われるものがあったので（phase が redirect だったのもあり）、この id で terraform import をしてみると成功！

ここまでできれば、だいたいの作業はおしまいです。

[Terraform のドキュメント](https://registry.terraform.io/providers/cloudflare/cloudflare/latest/docs/resources/ruleset#import)を参考に、ruleset のリソースを `terraform import` します。
`ruleset_id` は `"name": "default"` の id 値を指定すればOKです。

import が成功すると、state ファイルにリソース情報が書き込まれるので、terraform import 前後で state ファイルを比較し差分をterraformのコードに取り込めば完了です。

このあたりの terraform import の手順については本題から外れるので割愛します。
詳細な import の手順は他を参照してください。

## おわりに

これまでTerraformを用いてAWSのリソース構築もたくさんやってきましたが、CloudflareはAWSに比べて圧倒的に情報量が少なく、ハードルがかなり高いですね。

今回はリダイレクトルールを扱いましたが、これ以外でもよくわからなかった場合は、ブラウザ側でリソースを再生し `terraform import` でリソース定義をコピーしてくる。というのが良さそうです。

もうちょっと情報が充実してくれば…とは思いますが、CloudflareはAWSより小規模に使っている人が（自分も含めて）多い気がするので当分は難しいかもしれませんね😢
