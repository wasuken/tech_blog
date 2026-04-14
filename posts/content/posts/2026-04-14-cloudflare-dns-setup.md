---
title: "ドメイン購入後のやり残し作業 - メールとDNS設定編"
date: 2026-04-14T22:00:00+09:00
draft: false
tags: ["DNS", "Cloudflare", "SPF", "DMARC", "メール"]
categories: ["インフラ"]
---

wasutech.dev を公開したはいいが、メール周りを何もしていなかった。放置するとなりすましに悪用される可能性があるので、Cloudflare Email Routing・SPF・DMARCを設定した。

## Cloudflare Email Routing

自前のメールサーバーを建てるのはコストがかかる。Cloudflareには **Email Routing** という機能があり、`@wasutech.dev` 宛のメールを既存のGmailなどに転送できる。無料。

仕組みとしては、Cloudflareが自動でMXレコードを追加し、受信したメールを指定のアドレスへ転送する。

```bash
dig MX wasutech.dev
# → isaac.mx.cloudflare.net 等が返ってくる
```

MXレコードはドメインレベルの情報なので、転送先のメールアドレスは外部に公開されない。`dig` で見えるのはCloudflareのサーバーだけで、「どこに転送しているか」は誰にもわからない。

設定はCloudflareのダッシュボードから数クリックで完了する。

転送先のアクションは3種類から選べる。

| アクション | 動作 |
|---|---|
| Forward to email | 指定のメールアドレスへ転送 |
| Drop | 受信して即破棄。転送しない |
| Send to Worker | Cloudflare Workersで独自処理 |

今回はメールを受け取る必要がないため **Drop** に設定した。スパム対策にもなるし、転送先アドレスを管理する必要もない。Workers連携は受信メールをSlackに流したり、自動返信を実装したりする場合に使う。

## SPF - 送信元を証明するレコード

### SPFとは

**SPF（Sender Policy Framework）** は、「このドメインからメールを送信していいサーバーはどこか」を定義するDNSレコード。

なぜ必要か。メールのプロトコル（SMTP）は設計上、送信元アドレスを自由に詐称できる。つまり誰でも `admin@wasutech.dev` を名乗ってメールを送れる。SPFはこれを受信側が検証できるようにする仕組み。

受信側のメールサーバーは、届いたメールの送信元IPアドレスを確認し、そのドメインのSPFレコードに記載されたIPと照合する。一致しなければ怪しいメールとして処理できる。

### 設定したレコード

```
v=spf1 include:_spf.mx.cloudflare.net ~all
```

各要素の意味：

| 要素 | 意味 |
|---|---|
| `v=spf1` | SPFバージョン1 |
| `include:_spf.mx.cloudflare.net` | CloudflareのメールサーバーからのSMTPを許可 |
| `~all` | 上記以外は「疑わしい」扱い（ソフトフェイル） |

`~all` は疑わしいメールを迷惑メール扱いにする。`-all` にすると完全拒否になるが、DMARCと組み合わせて制御するのが一般的。

確認：

```bash
dig TXT wasutech.dev
# v=spf1 include:_spf.mx.cloudflare.net ~all
```

## DMARC - SPFの結果を使って何をするか決めるレコード

### DMARCとは

**DMARC（Domain-based Message Authentication, Reporting, and Conformance）** は、SPF（やDKIM）の検証結果に基づいて、受信側が「そのメールをどう扱うか」を指示するレコード。

SPFが「このメールは正規か？」を判定するなら、DMARCは「正規じゃなかったらどうすればいい？」をドメインオーナーが宣言する仕組み。

### ポリシー（`p=`）の違い

| ポリシー | 動作 |
|---|---|
| `p=none` | 何もしない。モニタリング専用 |
| `p=quarantine` | 迷惑メールフォルダに振り分け |
| `p=reject` | 完全に拒否。受信しない |

最初は `p=none` で様子を見てから上げていくのが定石だが、今回はメール送信自体しないブログ用ドメインなので最初から `p=reject` にした。

### `rua` - レポート送信先

DMARCには認証失敗のレポートを受け取るメールアドレスを指定できる（`rua=`）。Cloudflareが自動設定してくれるアドレスはハッシュ化されたもの。

```
rua=mailto:cea958224db542c0bcf04f319d159b3c@dmarc-reports.cloudflare.net
```

個人のメールアドレスは露出しない。

### 設定したレコード

DMARCは `_dmarc.wasutech.dev` というサブドメインのTXTレコードとして登録する。

```
v=DMARC1; p=reject; rua=mailto:cea958224db542c0bcf04f319d159b3c@dmarc-reports.cloudflare.net
```

確認：

```bash
dig TXT _dmarc.wasutech.dev
# v=DMARC1; p=reject; rua=mailto:...
```

## まとめ

| 設定 | 役割 |
|---|---|
| Email Routing | `@wasutech.dev` 宛メールをGmailへ転送 |
| SPF | 正規の送信元サーバーを定義 |
| DMARC | なりすましメールを拒否 |

ブログ用途で自分からメールを送信しないなら、この3つで十分。DKIMはメール送信時に必要になるが、今回は対象外。

Cloudflareのダッシュボードがほぼ自動でレコードを提案してくれるので、言われるがままに追加するだけで完了した。
