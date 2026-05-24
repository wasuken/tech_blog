---
title: "XBRL を自前パースして財務指標と監査情報を抽出する（Node.js）"
date: 2026-05-24T00:00:00+09:00
draft: true
tags: ["XBRL", "Node.js", "TypeScript", "財務分析", "EDINET", "個人開発"]
categories: ["個人開発"]
description: "EDINET から取得した XBRL（ZIP）を展開し、売上高・営業利益・監査法人名・GC注記・大株主情報を Node.js で抽出する。日本の XBRL 特有のタグ名前空間の扱いを解説。"
---

EDINET の書類には PDF の他に **XBRL** 形式のデータも含まれている。  
XBRL を使うと、売上高・営業利益・監査法人名・継続企業の前提に関する注記（GC注記）などを**構造化データとして**取り出せる。  
AI に PDF を読ませるのと組み合わせることで、定量＋定性のハイブリッド分析ができる。

## XBRL の取得

EDINET API で `type=1` を指定すると ZIP ファイルが返ってくる。  
中に `.xbrl` 拡張子のファイルが複数入っているので、目的のものを取り出す：

```typescript
import AdmZip from 'adm-zip'

const zipBuffer = await this.edinetClient.downloadXbrl(docId)
const zip = new AdmZip(zipBuffer)
const entries = zip.getEntries()

// jpcrp から始まるファイルが本体（企業情報報告書）
const xbrlEntry = entries.find(
  (e) => e.entryName.endsWith('.xbrl') && e.entryName.includes('jpcrp')
)
const xbrlContent = xbrlEntry?.getData().toString('utf-8') ?? ''
```

## XML パース

`fast-xml-parser` で XML をパース：

```typescript
import { XMLParser } from 'fast-xml-parser'

const parser = new XMLParser({
  ignoreAttributes: false,
  attributeNamePrefix: '@_',
  removeNSPrefix: true,  // 名前空間プレフィックスを除去
})
const parsed = parser.parse(xbrlContent)
```

`removeNSPrefix: true` がポイント。  
日本の XBRL は `jpcrp_cor:NetSales` のように名前空間プレフィックスが付いている。  
これを除去することで `NetSales` だけでアクセスできるようになる。

## 財務指標の抽出

売上高・営業利益などは `xbrli:context` の `contextRef` で期間を特定してから取る。  
当期（`CurrentYear` や `FilingDate` を含む context）を探す：

```typescript
function findValue(data: Record<string, unknown>, tagName: string): number | null {
  const val = data[tagName]
  if (Array.isArray(val)) {
    // 複数の context がある場合は当期のものを選ぶ
    const current = val.find(
      (v) => typeof v === 'object' && String(v?.['@_contextRef']).includes('Current')
    )
    return current ? Number(current['#text'] ?? current) : null
  }
  return val ? Number(val) : null
}
```

実際にはタグ名が企業によって微妙に違うことがあるため、候補タグを複数用意してフォールバックする。

## 監査情報の抽出

監査法人名と GC注記は以下のタグに入っている：

```typescript
// 監査法人名
const auditorName =
  data['NameOfIndependentAuditor'] ??
  data['AuditFirmName']

// GC注記の有無（"true" / "false" の文字列で入っていることがある）
const goingConcernRaw = data['NotesGoingConcernDoubtExistenceStringOfBoardOfDirectors']
const goingConcern = String(goingConcernRaw).length > 10  // 長い文章があれば注記あり
```

GC注記は `true/false` ではなく**注記本文テキストが入っている**ことが多いので、文字数で判定するのが現実的。

## 大株主の抽出

大株主情報は繰り返し要素として入っている：

```typescript
const shareholders = data['MajorShareholders']?.['ShareholderInformation'] ?? []
const list = Array.isArray(shareholders) ? shareholders : [shareholders]

return list.map((s) => ({
  name: String(s['NameOfShareholdersStringOfMajorShareholders'] ?? ''),
  shares: Number(s['NumberOfSharesHeld'] ?? 0),
  ratio: Number(s['ShareholdingRatio'] ?? 0),
}))
```

## まとめ

- XBRL は ZIP で取得して `.xbrl` ファイルを展開する
- `fast-xml-parser` の `removeNSPrefix: true` で名前空間を除去する
- 財務タグは企業によって揺れるのでフォールバック候補を持つ
- GC注記は `true/false` でなくテキスト本文が入っているので文字数で判定
- 大株主は繰り返し要素なので `Array.isArray` チェックが必要
