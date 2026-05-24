---
title: "Hono + TypeScript でクリーンアーキテクチャもどきを個人開発に持ち込む"
date: 2026-05-24T00:00:00+09:00
draft: true
tags: ["Hono", "TypeScript", "クリーンアーキテクチャ", "Node.js", "個人開発"]
categories: ["個人開発"]
description: "個人開発に過剰では？と思いつつも Domain / UseCase / Infrastructure の3層に分けたら、テストが書きやすくなったし外部 API の差し替えが楽になった話。"
---

個人開発に「クリーンアーキテクチャ」は過剰では？という気持ちはある。  
ただ実際にやってみたら、**テストが書きやすい・外部APIの差し替えが楽**という恩恵がちゃんとあった。  
Hono + TypeScript (ESM) でどう組んだかをメモしておく。

## ディレクトリ構成

```
backend/src/
├── domain/           # エンティティ・リポジトリ Interface
│   ├── entity/
│   └── repository/
├── usecase/          # ビジネスロジック
├── infrastructure/   # DB・外部API の実装
│   ├── postgres/
│   ├── edinet/
│   └── gemini/
├── api/              # Hono ルーター
└── job/              # JobRunner
```

## 依存の方向

```
api / job
    ↓
usecase          ← domain (Interface)
    ↓
infrastructure   → domain (Interface を実装)
```

`usecase` は `domain` の Interface にしか依存しない。  
`infrastructure` が Interface を実装する。これだけ守れば十分。

## Repository Interface の例

```typescript
// domain/repository/documentRepository.ts
export interface DocumentRepository {
  save(document: Document): Promise<void>
  exists(docId: string): Promise<boolean>
  findUnanalyzedOne(): Promise<Document | null>
  findById(docId: string): Promise<Document | null>
}
```

## PostgreSQL 実装

```typescript
// infrastructure/postgres/postgresDocumentRepository.ts
export class PostgresDocumentRepository implements DocumentRepository {
  constructor(private pool: Pool) {}

  async exists(docId: string): Promise<boolean> {
    const res = await this.pool.query(
      'SELECT 1 FROM documents WHERE doc_id = $1',
      [docId]
    )
    return (res.rowCount ?? 0) > 0
  }
  // ...
}
```

## UseCase でビジネスロジックを書く

```typescript
// usecase/fetchDocuments.ts
export class FetchDocumentsUseCase {
  constructor(
    private documentRepository: DocumentRepository,
    private edinetClient: EdinetClient
  ) {}

  async execute(date: string): Promise<void> {
    const results = await this.edinetClient.listDocuments(date)
    for (const res of results) {
      if (await this.documentRepository.exists(res.docID)) continue
      await this.documentRepository.save(/* ... */)
    }
  }
}
```

UseCase は `DocumentRepository` の Interface しか知らない。  
テストでは `vi.fn()` でモックしたオブジェクトを渡すだけ。

## テストが書きやすい

```typescript
describe('FetchDocumentsUseCase', () => {
  it('already existing docs are skipped', async () => {
    const mockRepo = {
      exists: vi.fn().mockResolvedValue(true),
      save: vi.fn(),
      // ...
    }
    const useCase = new FetchDocumentsUseCase(mockRepo, mockEdinetClient)
    await useCase.execute('2026-01-01')
    expect(mockRepo.save).not.toHaveBeenCalled()
  })
})
```

DB を立ち上げなくてもロジックをテストできる。

## Hono ルーターとの繋ぎ方

```typescript
// api/routes/documents.ts
const router = new Hono()

router.get('/', async (c) => {
  const docs = await documentRepo.findAll()
  return c.json(docs)
})

router.post('/:docId/analyze', async (c) => {
  const { docId } = c.req.param()
  const jobId = await jobRunner.createAndRun('analyze', { docId })
  return c.json({ jobId })
})
```

ルーターは Repository・JobRunner を直接受け取る。DI コンテナは使わず、`server.ts` でインスタンスを組み立てて渡している。個人開発ならこれで十分。

## まとめ

- Interface → UseCase → Infrastructure の依存方向を守るだけでかなり効果がある
- DI コンテナは不要。`server.ts` で手で組み立てる
- テストが書きやすくなるのは本当
- Hono は軽くて Bun / Node.js 両方で動くので個人開発に向いている
