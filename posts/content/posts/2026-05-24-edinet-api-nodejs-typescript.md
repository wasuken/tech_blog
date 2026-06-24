---
title: "EDINET API v2 で有価証券報告書を自動取得する（Node.js / TypeScript）"
date: 2026-05-24T00:00:00+09:00
draft: false
ai_assisted: true
tags: ["EDINET", "Node.js", "TypeScript", "API", "個人開発"]
categories: ["個人開発"]
description: "金融庁が公開する EDINET API v2 を使って、有価証券報告書・四半期報告書の一覧取得と PDF ダウンロードを TypeScript で実装する。型定義・レート制限対策・50KB スキップ判定・エラーハンドリングのポイントを解説。"
---

[EDINET](https://disclosure2.edinet-fsa.go.jp/WEEK0010.aspx) は金融庁が運営する電子開示システムで、上場企業が提出した有価証券報告書・四半期報告書などを無償で取得できる API を提供している。

個人開発の財務分析ツールを作るにあたって、この API を Node.js / TypeScript で叩いた際のポイントをまとめる。

## エンドポイント概要

ベース URL：`https://api.edinet-fsa.go.jp/api/v2`

| 用途 | エンドポイント |
|---|---|
| 書類一覧 | `GET /documents.json?date=YYYY-MM-DD&type=2` |
| PDF取得 | `GET /documents/{docID}?type=2` |
| XBRL取得 | `GET /documents/{docID}?type=1`（ZIP） |

API キーはクエリパラメータ `Subscription-Key` で渡す。[EDINET のサイト](https://api.edinet-fsa.go.jp/api/auth/index.aspx?mode=1) からアカウント登録すると発行される。**ハードコードせず `process.env.EDINET_API_KEY` から読む**のはもはや最低限のマナーといえる。

## レスポンスの型定義

まず API レスポンスに型をつける。`docID`（大文字）が EDINET 公式の表記：

```typescript
interface EdinetDocumentResponse {
  docID: string           // EDINET が振る書類ID
  docTypeCode: string | null
  secCode: string | null  // 証券コード。上場企業以外は null
  edinetCode: string
  filerName: string
  docDescription: string | null
  submitDateTime: string
}
```

クライアントクラスで型を明示しておくと、後続のフィルタリングや保存ロジックで補完が効いて安全になる。

## 書類一覧の取得とフィルタリング

`/documents.json` は指定日に提出されたすべての書類を返す。有価証券報告書だけを絞り込むには `docTypeCode` を見る：

- `120`：有価証券報告書
- `130`：訂正有価証券報告書
- `140`：四半期報告書

また `secCode` が `null` の書類は上場企業以外なのでスキップする。

```typescript
const filteredResults = results.filter((r: EdinetDocumentResponse) => {
  const typeCode = (r.docTypeCode ?? '').replace(/['"]/g, '')
  return (
    r.secCode != null &&
    (typeCode === '120' || typeCode === '130' || typeCode === '140')
  )
})
```

`docTypeCode` に余分なクォートが混入することがある（`"120"` のように入ってくる）ので `replace` で除去している。実際にハマった。

## PDF ダウンロードと 50KB スキップ

`type=2` で PDF が取得できる。ただし**極端に小さい PDF（50KB 未満）が稀に存在し**、そのままパース処理に渡すとエラーになる。

原因は諸説あるが、提出後の取り下げ・差し替えによって実質的に中身がない状態になっているケースや、EDINET 側の生成エラーで空のファイルが返るケースが確認されている。サイズチェックを挟んでスキップするのが現実的：

```typescript
try {
  const pdfBuffer = await this.edinetClient.downloadPdf(doc.docID)

  if (pdfBuffer.length < 50 * 1024) {
    logger.warn(`[FetchPdf] Skipping ${doc.docID}: too small (${pdfBuffer.length} bytes)`)
    // pdfFetched = true にして次回スキップ対象から外す
    await this.documentRepo.save({ ...doc, pdfFetched: true })
    return
  }

  await fs.writeFile(path.join(pdfDir, `${doc.docID}.pdf`), pdfBuffer)
  await this.documentRepo.save({ ...doc, pdfFetched: true })
  logger.info(`[FetchPdf] Saved ${doc.docID}`)

} catch (error) {
  // 失敗した docID をログに残し、次のサイクルで再試行させる
  logger.error(`[FetchPdf] Failed to download ${doc.docID}:`, error)
  throw error  // JobRunner 側で status='error' に更新される
}
```

エラーを `throw` することで JobRunner 側がジョブを `error` 状態にし、次のサイクルで再試行できる。失敗した `docID` はログに残るので、後から手動で再実行することも可能。

## レート制限対策

EDINET API には明示的な RPS 上限は公開されていないが、連続リクエストすると弾かれる。
スロットルキューを持たせるのが堅牢だが、シンプルにはループ内スリープでも十分：

```typescript
const apiKey = process.env.EDINET_API_KEY
if (!apiKey) throw new Error('EDINET_API_KEY is not set')

for (const doc of filteredDocs) {
  await new Promise<void>((r) => setTimeout(r, 3000))  // 最低 3 秒
  await downloadAndSave(doc, apiKey)
}
```

一応今の実装時点ではもっと大きな値を入れてる。

実運用では DB の `settings` テーブルにレート制限秒数を持たせ、UI から動的に変更できるようにしている。重い処理の日は10秒に上げるといった使い方ができる
。
## まとめ

- API キーは必ず環境変数から読む
- レスポンスに `EdinetDocumentResponse` 型をつけてコンパイル時に安全にする
- `docTypeCode` の余分なクォートに注意（`replace` で除去）
- `secCode` が `null` の書類は上場企業以外なのでフィルタ
- 50KB 未満の PDF は取り下げ等で空になっているケースがあるのでスキップ
- PDF ダウンロードは `try-catch` で囲み、失敗した `docID` をログに残して再試行できるようにする
