---
title: "Terraform学習1：variable/outputが分離しているのは何のためか"
date: 2026-07-25T09:00:00+09:00
draft: false
ai_assisted: true
tags: ["Terraform", "AWS", "インフラ", "設計"]
categories: ["技術"]
description: "variable/output/validation/stateの役割を整理しながら、パスワードをTerraformに渡さない設計の理由を掘り下げる"
---

Terraformを触り始めると、`variable`と`output`が真逆の役割を持っていることはすぐわかる。だが「なぜこの2つが分離しているのか」「なぜvalidationが型チェックと別枠なのか」まで理解しないと、秘匿情報をうっかりstateに残す事故につながる。今回は自分のTerraform構成（`variables.tf`, `outputs.tf`, `versions.tf`のbackend部分）を題材に、設計意図を整理する。

## variableは「設定とロジックの分離」のための入口

`variable`は、値をリソース定義本体に直書きせず、外部から注入するための仕組みだ。値を変えるたびにリソース定義そのものを編集しなくて済むようにする、という目的がある。

値の入力元は主に4つある。

- `default`（ブロック内で指定）
- `terraform.tfvars`
- 環境変数（`TF_VAR_xxx`）
- `-var`オプション（CLI実行時）

### defaultの有無で意味が変わる

| | 意味 |
|---|---|
| defaultあり | 共通・変更頻度が低い値。省略可能（例: `aws_region`, `project_name`） |
| defaultなし | 環境固有・必須の値。指定しないと`plan`/`apply`時にエラー（例: `budget_alert_email`, `github_org`） |

**defaultの有無は「省略可能かどうか」の表明であり、そのままその変数の性格（共通設定か、環境固有の必須値か）を表している。**

### validationは型チェックとは別レイヤーの検証

`variable`の`type`は形式のチェックしかしない。`type = string`は「文字列であること」しか保証せず、「その文字列が正しい値かどうか」は見ない。

そこを埋めるのが`validation`ブロックだ。値の中身・意味をチェックする。例えばSSMパラメータ名は`/`始まりである必要がある、EBSルートボリュームは8GiB以上必要、といった業務ルールをここに書く。

```hcl
variable "image_id" {
  type        = string
  description = "The ID of the machine image (AMI) to use for the server."

  validation {
    condition     = length(var.image_id) > 4 && substr(var.image_id, 0, 4) == "ami-"
    error_message = "The image_id value must be a valid AMI ID, starting with \"ami-\"."
  }
}
```

`validation`はapply前、ローカルの`plan`段階でエラーを弾ける。**無駄なapply実行を未然に防げる**のが最大の利点で、クラウド側にリクエストが飛ぶ前に「その値はそもそもおかしい」と教えてくれる。

## パスワードを`variable`に直接渡さない設計にする理由

自分の構成では、`db_root_password_parameter_name`のように、パスワードの値そのものではなく「SSM Parameter Store上のパラメータ名」を変数として渡している。最初は「なぜ`sensitive = true`をつけたvariableでよくないのか」と思ったが、理由はリスクの経路が2つあり、しかもそのどちらも`sensitive = true`では防げないことにある。

### リスク経路1: stateファイルへの平文記録

Terraformに渡した値は、`apply`後に`terraform.tfstate`へ自動的に平文で記録される。`sensitive = true`はCLI出力（ログやplan結果の表示）を隠すだけで、**state本体の中身は変わらない**。これは`variable`の仕組みそのものでは防げない領域になる。

### リスク経路2: tfvarsのGit事故

`terraform.tfvars`に秘匿値を直書きすると、`.gitignore`漏れなどでコミット履歴に残るリスクがある。一度コミットされた履歴は、後から`.gitignore`に追加しても消えない。

### 対策: 実体を渡さない設計

パスワードの実体をTerraformに一切渡さず、「SSM Parameter Store上のパラメータ名」という間接参照だけを渡す設計にすることで、上記2つのリスク経路を両方回避できる。stateに残るのはパラメータ名という文字列だけで、パスワードそのものはSSM側で管理される。

**`variable`の仕組み自体が無意味というわけではない。「stateに残っても問題ない値」と「stateに残ってはいけない値」を、渡す中身のレベルで使い分けているだけだ。**

### stateに残っても許容される値の考え方

`budget_alert_email`のような値も、技術的にはstateに残る。これは例外ではなく通常の挙動だ。しかし「stateに残る」＝即アウトではなく、**漏洩時の被害の大きさ**で対応レベルを判断すればいい。

| 種類 | 漏洩時の被害 | 対応 |
|---|---|---|
| パスワード | 即座に侵入経路になる | 実体を持たせない設計が必要 |
| メールアドレス等 | 間接的・軽微 | stateに残る前提で許容 |

前提として、state自体も多層防御されている。S3保存時のサーバーサイド暗号化（`encrypt = true`）と、IAMによるバケットへのアクセス権限の限定（Private運用）だ。この土台があるからこそ、「メールアドレス程度ならstateに残ってよい」という判断が成立する。

## outputは「Terraformの外の世界」への橋渡し

`output`はTerraformが作ったリソースの実際の値を外に出す仕組みで、公式ドキュメントではユースケースが整理されている。

1. `terraform apply`後にCLIへ表示し、`terraform output`コマンドで個別取得する
2. 子モジュールの属性を親モジュールに公開する（module分割時）
3. `terraform_remote_state`で別のTerraform構成（state）から参照する（state分割時）

自分の構成はmodule分割・state分割をしていない単一ディレクトリ構成のため、ユースケース1のみが活きている。2と3は現状該当しない。

### descriptionの有無が実は運用上の意味を持っていた

自分のoutputのうち、`description`がついているのは`github_actions_role_arn`, `ec2_instance_id`, `ec2_elastic_ip`の3つだけだった。最初は単に「書き忘れ」かと思っていたが、見直すとこの3つには共通点がある。

**Terraformの外の世界（GitHub Secrets/Variables、Cloudflareの管理画面など）に、人間が手動で値を持ち出して使うもの**だけに`description`がついていた。実際の運用では、ARNはGitHub Secretsへ、インスタンスIDはGitHub Variablesへ手動転記している。

Secrets/Variablesの使い分けも、漏洩時の実害の大小と変更頻度で判断している。

| output | 転記先 | 理由 |
|---|---|---|
| `github_actions_role_arn` | GitHub Secrets | 変更頻度が低く、漏洩時のリスクが比較的高い |
| `ec2_instance_id` | GitHub Variables | インスタンス作り直しで変わりうる、見えても実害が薄い |

一方、`description`が無い残り8つは、Terraform管理下の中で完結する確認・トラブルシュート用途にとどまる。**「人間が手動で持ち出して使う値かどうか」が、descriptionを書くかどうかの実質的な判断基準になっていた。**

## backend "s3"とstate管理

`versions.tf`内の`terraform {}`ブロックで、state保存先をS3バケットに設定している。

```hcl
terraform {
  backend "s3" {
    bucket       = "myproject-terraform-state"
    key          = "terraform.tfstate"
    region       = "ap-northeast-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

`use_lockfile = true`は<cite index="19-1">Terraformのs3バックエンドに用意されているオプション引数で、デフォルトは無効になっている</cite>。これを有効にすると、DynamoDBテーブルを使わずにS3だけで排他制御ができる。<cite index="17-1">仕組みとしては、S3のconditional writes（PutObjectのIf-None-Matchヘッダ）を使ってロックファイルを作成し、すでに同名のオブジェクトが存在する場合はS3側がエラーを返すことで競合を防いでいる</cite>。DynamoDBの管理コストを削減できるため、リソース最小化の方針に合っている。

`use_lockfile`は<cite index="10-1">Terraform 1.10で実験的機能として導入され、1.11でデフォルトのオプション機能として正式化された</cite>ものなので、古いTerraformバージョンでは使えない点に注意したい。

### 鶏卵問題への対処

backend用のS3バケット自体もTerraformで管理する対象のため、「バケットが存在しない状態でbackendをS3に向けると矛盾する」という鶏と卵の問題が発生する。対処は3ステップ。

1. 初回applyは`backend "s3"`ブロックをコメントアウトし、ローカルstateで実行する（S3バケットを含むリソース一式を作成）
2. バケット作成後、`backend "s3"`ブロックを有効化する
3. `terraform init -migrate-state`でローカルstateをS3へ移行する

## まとめ

| 要素 | 役割 | 注意点 |
|---|---|---|
| `variable` | 設定とロジックの分離 | defaultの有無が値の性格を表す |
| `validation` | 値の意味的な正しさをapply前に検証 | 型チェックとは別レイヤー |
| `output` | Terraform管理下の値を外の世界へ橋渡し | descriptionの有無が運用上の意味を持つことがある |
| `backend "s3"` | state保存先の一元化 | 鶏卵問題があるため初回applyはローカルstateで |

**variableとoutputの分離は単なる入出力の区別ではなく、「stateに残してよい値」と「stateに残してはいけない値」の境界線を設計する仕組みでもある。**パスワードのような秘匿値は、そもそもTerraformに実体を渡さない設計にすることで、stateという単一の弱点を経由するリスクごと消せる。

## 参考

- [Backend Type: s3 | Terraform | HashiCorp Developer](https://developer.hashicorp.com/terraform/language/backend/s3)
- [Output block reference for the Terraform configuration language | Terraform | HashiCorp Developer](https://developer.hashicorp.com/terraform/language/block/output)
- [The terraform_remote_state Data Source | Terraform | HashiCorp Developer](https://developer.hashicorp.com/terraform/language/state/remote-state-data)
- [variable block reference for the Terraform configuration language | Terraform | HashiCorp Developer](https://developer.hashicorp.com/terraform/language/block/variable)
- [Validate your infrastructure in Terraform's configuration language | Terraform | HashiCorp Developer](https://developer.hashicorp.com/terraform/language/validate)
