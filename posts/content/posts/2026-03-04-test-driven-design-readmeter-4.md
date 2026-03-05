---
title: "テスト前提で設計したWebアプリのハンズオン - 読書管理アプリ その4"
date: 2026-03-05T00:50:00+09:00
tags: ["Next.js", "Prisma", "TypeScript", "DDD", "テスト設計"]
categories: ["開発"]
---

## はじめに

[Part1](./part1) でValueObject・Policy・Entityを作り、[Part2](./part2) でServiceをDI＋Mockテストで固めた。
[Part3](./part3) でPrismaを繋ぎ、`GET /api/books` と `POST /api/books` を動かした。

Part4では残りのエンドポイントを実装する。やることは3つ。

1. カスタム例外クラスの導入
2. `PATCH /api/books/:id/start` と `PATCH /api/books/:id/complete` の実装
3. `DELETE /api/books/:id` の実装

そして「**エラー種別ごとにHTTPステータスを整理する**」という設計判断を掘り下げる。

また、Part3でPrisma v7特有のセットアップをしたが、v6以前と何が変わったのかをここで整理しておく。

---

## 0. Prisma v7で何が変わったか

Part3でPrismaをセットアップしたとき、v6以前と比べていくつか「見慣れない書き方」が必要だった。
v7は破壊的変更が多く、ネット上のv6時代の記事を参考にすると詰まる箇所がある。
ここで整理しておく。

参考: [Upgrade to Prisma ORM 7 | Prisma Documentation](https://www.prisma.io/docs/orm/more/upgrade-guides/upgrading-versions/upgrading-to-prisma-7)

### generator の変更

```prisma
// ❌ v6以前
generator client {
  provider = "prisma-client-js"
}

// ✅ v7
generator client {
  provider = "prisma-client"
  output   = "../src/generated/prisma"
}
```

v7では `prisma-client-js` が廃止され `prisma-client` に変わった。
Rustベースのエンジンを廃止しTypeScriptネイティブになったことに伴う変更だ。
また `output` が必須になり、`node_modules` への自動生成はなくなった。

importパスもこれに伴って変わる。

```typescript
// ❌ v6以前
import { PrismaClient } from '@prisma/client'

// ✅ v7
import { PrismaClient } from '@/generated/prisma/client'
```

### driver adapterが必須になった

v7では `PrismaClient` の初期化に必ずdriver adapterを渡す必要がある。
SQLiteの場合は `@prisma/adapter-better-sqlite3` を使う。

```typescript
// ❌ v6以前
const prisma = new PrismaClient()

// ✅ v7
import { PrismaBetterSqlite3 } from '@prisma/adapter-better-sqlite3'
const adapter = new PrismaBetterSqlite3({ url: process.env.DATABASE_URL || 'file:./dev.db' })
const prisma = new PrismaClient({ adapter })
```

参考: [Prisma ORM Quickstart with SQLite](https://www.prisma.io/docs/getting-started/prisma-orm/quickstart/sqlite)

### prisma.config.ts の導入

v7ではCLI設定を `prisma.config.ts` に集約する方式になった。
`prisma migrate dev` などのコマンドはここからDB接続情報を読む。

```typescript
// prisma.config.ts
import 'dotenv/config'
import { defineConfig } from 'prisma/config'

export default defineConfig({
  schema: 'prisma/schema.prisma',
  migrations: {
    path: 'prisma/migrations',
  },
  datasource: {
    url: process.env['DATABASE_URL'],
  },
})
```

`schema.prisma` の `datasource` ブロックにあった `url` はここに移す。
v7では `schema.prisma` の `url` はdeprecatedになり、`prisma.config.ts` が優先される。

### HMR対策のglobalThisキャッシュ

Next.jsの開発サーバーはHMR（Hot Module Replacement）でモジュールを再評価する。
毎回 `new PrismaClient()` するとコネクションが枯渇する。
`globalThis` にキャッシュして使い回す。

```typescript
// src/lib/prisma.ts
import { PrismaClient } from '@/generated/prisma/client'
import { PrismaBetterSqlite3 } from '@prisma/adapter-better-sqlite3'

const adapter = new PrismaBetterSqlite3({
  url: process.env.DATABASE_URL || 'file:./dev.db',
})

const globalForPrisma = globalThis as unknown as { prisma?: PrismaClient }

export const prisma =
  globalForPrisma.prisma ?? new PrismaClient({ adapter })

if (process.env.NODE_ENV !== 'production') {
  globalForPrisma.prisma = prisma
}
```

参考: [Prisma Client in Next.js | Prisma Documentation](https://www.prisma.io/docs/guides/other/troubleshooting-orm/help-articles/nextjs-prisma-client-dev-practices)

---

## 1. カスタム例外クラスの導入

これまで `BookShelfService` では `throw new Error(...)` を使っていた。
Route Handlerで「このエラーは404か、400か、422か」を判定するには `instanceof` チェックが必要だ。
エラーメッセージの文字列に依存した判定は壊れやすい。

`src/errors/AppError.ts` を作る。

```typescript
export class NotFoundError extends Error {
  constructor(message: string) {
    super(message);
    this.name = "NotFoundError";
  }
}

export class DomainError extends Error {
  constructor(message: string) {
    super(message);
    this.name = "DomainError";
  }
}
```

シンプルに `Error` を継承するだけだ。
今回 `DomainError` は使わないが、将来のドメインルール違反（積読→読了の直接遷移など）のために定義しておく。

### BookShelfService を更新する

`NotFoundError` を `import` して、 `throw new Error` を置き換える。

```typescript
import { NotFoundError } from "../errors/AppError";

// 変更前
if (!book) throw new Error(`Book not found: ${id}`);

// 変更後
if (!book) throw new NotFoundError(`Book not found: ${id}`);
```

既存テストのメッセージ検証（`"Book not found: not-exist"`）はそのまま通る。
さらに `toBeInstanceOf(NotFoundError)` でクラスも検証するテストを追加した。

---

## 2. エラーハンドリングヘルパー

Route Handlerを3本書くと、エラーハンドリングが重複する。
`src/lib/handleError.ts` に共通ロジックを切り出す。

```typescript
import { NextResponse } from "next/server";
import { NotFoundError, DomainError } from "../errors/AppError";

export function handleError(e: unknown): NextResponse {
  if (e instanceof NotFoundError) {
    return NextResponse.json({ error: e.message }, { status: 404 });
  }
  if (e instanceof DomainError) {
    return NextResponse.json({ error: e.message }, { status: 422 });
  }
  if (e instanceof Error) {
    return NextResponse.json({ error: e.message }, { status: 400 });
  }
  return NextResponse.json({ error: "Internal Server Error" }, { status: 500 });
}
```

エラーの優先順位は以下の通り。

| 例外クラス | HTTPステータス | 使いどころ |
|---|---|---|
| `NotFoundError` | 404 | リソースが存在しない |
| `DomainError` | 422 | ビジネスルール違反 |
| `Error` | 400 | バリデーションエラー（ISBNが不正など） |
| その他 | 500 | 予期せぬエラー |

`DomainError` が 422（Unprocessable Entity）なのは、リクエスト自体は正しい形式だが業務上処理できない場合に使う慣習に倣った。
参考: [RFC 9110 Section 15.5.21 422 Unprocessable Content](https://www.rfc-editor.org/rfc/rfc9110#section-15.5.21)

---

## 3. PATCH /api/books/[id]/start の実装

`src/app/api/books/[id]/start/route.ts` を作る。

```typescript
import { NextResponse } from "next/server";
import { prisma } from "@/lib/prisma";
import { PrismaBookRepository } from "@/repository/PrismaBookRepository";
import { BookShelfService } from "@/service/BookShelfService";
import { handleError } from "@/lib/handleError";

function buildService(): BookShelfService {
  const repo = new PrismaBookRepository(prisma);
  return new BookShelfService(repo);
}

export async function PATCH(
  _req: Request,
  { params }: { params: Promise<{ id: string }> },
) {
  try {
    const { id } = await params;
    const service = buildService();
    const book = await service.startReading(id);

    return NextResponse.json({
      id: book.id,
      title: book.title,
      isbn: book.isbn.value,
      status: book.status,
      rating: book.rating?.value ?? null,
    });
  } catch (e) {
    return handleError(e);
  }
}
```

### Next.js 15以降の params は Promise

**注意点**：Next.js 15以降、Dynamic Route の `params` は `Promise` になった。
> 参考: https://nextjs.org/docs/app/api-reference/file-conventions/route

```typescript
// ❌ Next.js 14以前のパターン（15では動かない）
export async function PATCH(
  _req: Request,
  { params }: { params: { id: string } },
) {
  const { id } = params; // エラー: params should be awaited
}

// ✅ Next.js 15以降のパターン
export async function PATCH(
  _req: Request,
  { params }: { params: Promise<{ id: string }> },
) {
  const { id } = await params; // awaitが必要
}
```

---

## 4. PATCH /api/books/[id]/complete の実装

`src/app/api/books/[id]/complete/route.ts` を作る。
こちらはリクエストボディに `{ "rating": number }` を受け取る（省略可）。

```typescript
import { NextRequest, NextResponse } from "next/server";
import { prisma } from "@/lib/prisma";
import { PrismaBookRepository } from "@/repository/PrismaBookRepository";
import { BookShelfService } from "@/service/BookShelfService";
import { handleError } from "@/lib/handleError";

function buildService(): BookShelfService {
  const repo = new PrismaBookRepository(prisma);
  return new BookShelfService(repo);
}

export async function PATCH(
  req: NextRequest,
  { params }: { params: Promise<{ id: string }> },
) {
  try {
    const { id } = await params;
    const body = await req.json().catch(() => ({}));
    const rating: number | undefined =
      typeof body.rating === "number" ? body.rating : undefined;

    const service = buildService();
    const book = await service.completeReading(id, rating);

    return NextResponse.json({
      id: book.id,
      title: book.title,
      isbn: book.isbn.value,
      status: book.status,
      rating: book.rating?.value ?? null,
    });
  } catch (e) {
    return handleError(e);
  }
}
```

### ポイント：body のパースに `.catch(() => ({}))` を使う理由

`rating` はオプションなのでボディが空のリクエストも受け付けたい。
`req.json()` はボディが空だと例外を投げる。
`.catch(() => ({}))` で空ボディを安全に `{}` に変換している。

---

## 5. DELETE /api/books/[id] の実装

`src/app/api/books/[id]/route.ts` を作る。

```typescript
import { NextResponse } from "next/server";
import { prisma } from "@/lib/prisma";
import { PrismaBookRepository } from "@/repository/PrismaBookRepository";
import { BookShelfService } from "@/service/BookShelfService";
import { handleError } from "@/lib/handleError";

function buildService(): BookShelfService {
  const repo = new PrismaBookRepository(prisma);
  return new BookShelfService(repo);
}

export async function DELETE(
  _req: Request,
  { params }: { params: Promise<{ id: string }> },
) {
  try {
    const { id } = await params;
    const service = buildService();
    await service.removeBook(id);

    return new NextResponse(null, { status: 204 });
  } catch (e) {
    return handleError(e);
  }
}
```

削除成功時は `204 No Content` を返す。
レスポンスボディは不要なので `NextResponse.json({})` ではなく `new NextResponse(null, { status: 204 })` を使う。

---

## 6. ディレクトリ構成の確認

Part4完了時点の新規追加ファイルは以下の通り。

```
src/
  errors/
    AppError.ts                          # NotFoundError / DomainError
  lib/
    handleError.ts                       # Route Handler共通エラーハンドラ
  app/api/books/
    [id]/
      route.ts                           # DELETE /api/books/:id
      start/
        route.ts                         # PATCH /api/books/:id/start
      complete/
        route.ts                         # PATCH /api/books/:id/complete
```

---

## 7. 動作確認

```bash
npm run dev
```

```bash
# 本を追加
curl -X POST http://localhost:3000/api/books \
  -H "Content-Type: application/json" \
  -d '{"id":"1","title":"Clean Code","isbn":"9784048860000"}'

# 積読→読中
curl -X PATCH http://localhost:3000/api/books/1/start

# 読中→読了（評価あり）
curl -X PATCH http://localhost:3000/api/books/1/complete \
  -H "Content-Type: application/json" \
  -d '{"rating":5}'

# 読了確認
curl http://localhost:3000/api/books

# 本を削除
curl -X DELETE http://localhost:3000/api/books/1

# 削除後確認（空配列）
curl http://localhost:3000/api/books
```

存在しないIDへのリクエストは404が返ることも確認しておく。

```bash
# 存在しない本へのアクセス
curl -X PATCH http://localhost:3000/api/books/not-exist/start
# {"error":"Book not found: not-exist"} 404
```

テストも確認する。

```bash
npm run ci
# tsc --noEmit && vitest run && eslint
```

既存13テストに加え、`NotFoundError` のインスタンス検証テストが2件追加されて計15テスト、全パスの状態になる。

---

## 所感：例外クラスの設計は「誰が責任を持つか」で決まる

今回 `NotFoundError` と `DomainError` の2クラスを導入した。

```
BookShelfService → NotFoundError を throw
Book.changeStatus → Error を throw（Policyが弾いた）
BookShelfService → DomainError に変換する選択肢もあった
```

`Book.changeStatus` が投げる `Error` をそのまま通過させた設計にした。
Route Handlerの `handleError` では `instanceof Error` で400として受け取る。
「ISBNが不正」「状態遷移が不正」はどちらも400（クライアントの入力起因）として統一した。

将来的に「積読→読了の遷移エラーは422にしたい」という要件が出たら、その時点で `Book.changeStatus` が `DomainError` を投げるように変える。
今は必要ないのでやらない。**YAGNIの原則**だ。

---

## まとめ

Part4でやったこと：

- Prisma v7の主要な破壊的変更（generator・driver adapter・prisma.config.ts）を整理した
- `NotFoundError` / `DomainError` カスタム例外クラスを導入した
- `handleError` ヘルパーでRoute Handler間のエラー処理を共通化した
- `PATCH /api/books/:id/start` で積読→読中のステータス変更を実装した
- `PATCH /api/books/:id/complete` で読中→読了（評価オプション）を実装した
- `DELETE /api/books/:id` で本の削除を実装した
- テストは13→15件に増え、引き続き全パス

次はフロントエンドの簡易UIを繋ぐか、ReviewServiceを実装するか予定。
