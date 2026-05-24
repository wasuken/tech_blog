---
title: "EDINET API v2 で有価証券報告書を自動取得する（Node.js / TypeScript）"
date: 2026-05-24T00:00:00+09:00
draft: true
tags: ["EDINET", "Node.js", "TypeScript", "API", "個人開発"]
categories: ["個人開発"]
description: "金融庁が公開する EDINET API v2 を使って、有価証券報告書・四半期報告書の一覧取得と PDF ダウンロードを TypeScript で実装する。レート制限対策と 50KB スキップ判定のポイントを解説。"
---

[EDINET](https://edinet.fsa.go.jp/) は金融庁が運営する電子開示システムで、上場企業が提出した有価証券報告書・四半期報告書などを無償で取得できる API を提供している。

個人開発の財務分析ツールを作るにあたって、この API を Node.js / TypeScript で叩いた際のポイントをまとめる。

## エンドポイント概要

ベース URL：`https://api.edinet-fsa.go.jp/api/v2`

| 用途 | エンドポイント |
|---|---|
| 書類一覧 | `GET /documents.json?date=YYYY-MM-DD&type=2` |
| PDF取得 | `GET /documents/{docId}?type=2` |
| XBRL取得 | `GET /documents/{docId}?type=1`（ZIP） |

API キーはクエリパラメータ `Subscription-Key` で渡す。[EDINETのサイト](https://api.edinet-fsa.go.jp/) からアカウント登録すると発行される。

## 書類一覧の取得とフィルタリング

`/documents.json` は指定日に提出されたすべての書類を返す。有価証券報告書だけを絞り込むには `docTypeCode` を見る：

- `120`：有価証券報告書
- `130`：訂正有価証券報告書
- `140`：四半期報告書

また `secCode`（証券コード）が存在しない書類は上場企業以外なのでスキップする。

```typescript
const filteredResults = results.filter((r) => {
  const typeCode = (r.docTypeCode || '').replace(/['"]/g, '')
  return (
    r.secCode &&
    (typeCode === '120' || typeCode === '140' || typeCode === '130')
  )
})
```

`docTypeCode` に余分なクォートが混入することがあるので `replace` で除去している（実際にハマった）。

## PDF ダウンロードと 50KB スキップ

`type=2` で PDF が取得できる。ただし**空に近い PDF（50KB 未満）が稀に存在し**、そのままパースに渡すとエラーになる。サイズチェックを挟む：

```typescript
const buffer = await this.downloadPdf(docId)
if (buffer.length < 50 * 1024) {
  logger.warn(`Skipping ${docId}: PDF too small (${buffer.length} bytes)`)
  return
}
```

## レート制限対策

EDINET API には明示的な上限は書かれていないが、連続リクエストすると弾かれる。  
ループ内で **3〜10 秒のスリープ**を挟むのが安全：

```typescript
for (const doc of filteredDocs) {
  await new Promise((r) => setTimeout(r, 3000))
  const buffer = await this.downloadPdf(doc.docId)
  // ...
}
```

実運用では `settings` テーブルにレート制限秒数を持たせて、UI から変更できるようにしている。

## まとめ

- `docTypeCode` の余分なクォートに注意
- `secCode` がない書類は上場企業以外なのでフィルタ
- PDF は 50KB 未満をスキップする
- ループ内に最低 3 秒のスリープを入れる
