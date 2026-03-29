---
title: "テスト前提で設計したWebアプリのハンズオン - 読書管理アプリ その3"
date: 2026-03-04T17:50:00+09:00
tags: ["Next.js", "Prisma", "TypeScript", "DDD", "テスト設計"]
categories: ["開発"]
---

## はじめに

[Part1](./part1) でValueObject・Policy・Entityを作り、[Part2](./part2) でServiceをDI＋Mockテストで固めた。
累計13テスト、全パスの状態だ。

Part3ではいよいよDBを繋ぐ。やることは3つ。

1. Prismaセットアップ＋スキーマ定義
2. `PrismaBookRepository` 実装（ドメインオブジェクトへの変換）
3. Route HandlerでDIを組み立てる

そして最後に「**テストを書かない層を意図的に決める**」という話をする。

---

## 1. Prismaセットアップ

```bash
npm install prisma @prisma/client
npx prisma init --datasource-provider sqlite
```

`prisma/schema.prisma` に Bookモデルを定義する。

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "sqlite"
  url      = env("DATABASE_URL")
}

model Book {
  id     String  @id
  title  String
  isbn   String
  status String
  rating Int?
}
```

`User` と `Review` はまだDBに持たない。Book の CRUD が動けば Part3 のゴール。

### なぜ `status` を String で持つのか

Prismaは SQLiteで enum をネイティブサポートしていない。
そのため `status String` で持ち、取り出し時に `as ReadingStatus` でキャストする。

```typescript
// DBレコード → ドメインオブジェクト変換時
record.status as ReadingStatus
```

不正値が入るリスクはある。ただしその責任はRepository層の内側に閉じている。
外のService層・ドメイン層は `ReadingStatus` 型として受け取るので影響を受けない。
この割り切りはアーキテクチャ上の判断であり、ここに `ReadingStatusPolicy` で検証を追加することもできる。今回はシンプルさを優先した。

マイグレーションを実行する。

```bash
npx prisma migrate dev --name init
```

`.env` に `DATABASE_URL="file:./dev.db"` が自動生成されていることを確認する。

---

## 2. PrismaBookRepository実装

`src/repository/PrismaBookRepository.ts` を新規作成する。
`BookRepository` interface を満たせばよく、Serviceはこの実装クラスを直接知らない。

```typescript
import { PrismaClient, Book as PrismaBook } from "@prisma/client";
import { BookRepository } from "./BookRepository";
import { Book } from "../domain/entity/Book";
import { ISBN } from "../domain/valueobject/ISBN";
import { Rating } from "../domain/valueobject/Rating";
import { ReadingStatus } from "../domain/valueobject/ReadingStatus";

export class PrismaBookRepository implements BookRepository {
  constructor(private readonly prisma: PrismaClient) {}

  async save(book: Book): Promise<void> {
    await this.prisma.book.upsert({
      where: { id: book.id },
      update: this.toRecord(book),
      create: { id: book.id, ...this.toRecord(book) },
    });
  }

  async findById(id: string): Promise<Book | null> {
    const record = await this.prisma.book.findUnique({ where: { id } });
    if (!record) return null;
    return this.toDomain(record);
  }

  async findAll(): Promise<Book[]> {
    const records = await this.prisma.book.findMany();
    return records.map((r) => this.toDomain(r));
  }

  async delete(id: string): Promise<void> {
    await this.prisma.book.delete({ where: { id } });
  }

  // ドメインオブジェクト → DBレコード用プレーンオブジェクト
  private toRecord(book: Book) {
    return {
      title: book.title,
      isbn: book.isbn.value,
      status: book.status,
      rating: book.rating?.value ?? null,
    };
  }

  // DBレコード → ドメインオブジェクト
  private toDomain(record: PrismaBook): Book {
    return new Book(
      record.id,
      record.title,
      new ISBN(record.isbn),
      record.status as ReadingStatus,
      record.rating !== null ? new Rating(record.rating) : null,
    );
  }
}
```

### ポイント：変換ロジックはRepositoryの内側に閉じる

`toDomain` と `toRecord` はこのクラスの `private` メソッドにしている。
外部から変換ロジックを操作できない設計だ。

- Serviceは `Book` ドメインオブジェクトしか知らない
- Prismaの `Book`（DBスキーマ由来の型）はRepository内にしか登場しない
- DBスキーマが変わっても、変更箇所はここだけで済む

### `save` に upsert を使う理由

`addBook`（INSERT相当）と `startReading`/`completeReading`（UPDATE相当）が同じ `save` メソッドに集約されている。
Service層はDBの「新規か更新か」を気にしない。
Repositoryが `upsert` で吸収する。

```
Service: save(book) を呼ぶだけ
Repository: id存在チェックして INSERT or UPDATE を判断する
```

この責任分離がDIの恩恵の一つだ。

---

## 3. PrismaClient のシングルトン管理

Next.jsの開発環境ではHMR（Hot Module Replacement）のたびにモジュールが再評価される。
`new PrismaClient()` が毎回走ると接続が枯渇する。

公式が推奨するパターンで回避する。
> 参考: https://www.prisma.io/docs/orm/more/troubleshooting/nextjs

```typescript
// src/lib/prisma.ts
import { PrismaClient } from "@prisma/client";

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient };

export const prisma = globalForPrisma.prisma ?? new PrismaClient();

if (process.env.NODE_ENV !== "production") {
  globalForPrisma.prisma = prisma;
}
```

`globalThis` に一度生成したインスタンスを保持し、HMR後も使い回す。
`production` では毎回 `new PrismaClient()` で問題ない（HMRが走らないため）。

---

## 4. Route HandlerでDIを組み立てる

`src/app/api/books/route.ts` を作る。
ここが唯一の「組み立て場所」だ。

```typescript
import { NextRequest, NextResponse } from "next/server";
import { prisma } from "@/lib/prisma";
import { PrismaBookRepository } from "@/repository/PrismaBookRepository";
import { BookShelfService } from "@/service/BookShelfService";

function buildService(): BookShelfService {
  const repo = new PrismaBookRepository(prisma);
  return new BookShelfService(repo);
}

export async function GET() {
  try {
    const service = buildService();
    const books = await service.getAll();

    const body = books.map((b) => ({
      id: b.id,
      title: b.title,
      isbn: b.isbn.value,
      status: b.status,
      rating: b.rating?.value ?? null,
    }));

    return NextResponse.json(body);
  } catch (e) {
    console.error(e);
    return NextResponse.json({ error: "Internal Server Error" }, { status: 500 });
  }
}

export async function POST(req: NextRequest) {
  try {
    const { id, title, isbn } = await req.json();

    if (!id || !title || !isbn) {
      return NextResponse.json(
        { error: "id, title, isbn は必須です" },
        { status: 400 },
      );
    }

    const service = buildService();
    const book = await service.addBook(id, title, isbn);

    return NextResponse.json(
      {
        id: book.id,
        title: book.title,
        isbn: book.isbn.value,
        status: book.status,
        rating: book.rating?.value ?? null,
      },
      { status: 201 },
    );
  } catch (e) {
    if (e instanceof Error) {
      return NextResponse.json({ error: e.message }, { status: 400 });
    }
    return NextResponse.json({ error: "Internal Server Error" }, { status: 500 });
  }
}
```

### 依存の流れ

```
Route Handler
  └─ buildService()
       ├─ PrismaClient（インフラ）
       ├─ PrismaBookRepository（BookRepository interfaceを実装）
       └─ BookShelfService（BookRepository interfaceだけを知っている）
```

Service は interface しか知らない。
Prismaに関する知識はRoute HandlerとRepositoryの2箇所だけに集中している。

---

## 5. 動作確認

```bash
npx prisma migrate dev --name init
npm run dev
```

```bash
# 一覧取得（初期は空配列）
curl http://localhost:3000/api/books

# 本を追加
curl -X POST http://localhost:3000/api/books \
  -H "Content-Type: application/json" \
  -d '{"id":"1","title":"Clean Code","isbn":"9784048860000"}'

# 追加後に一覧取得
curl http://localhost:3000/api/books
```

期待するレスポンス例：

```json
[
  {
    "id": "1",
    "title": "Clean Code",
    "isbn": "9784048860000",
    "status": "Unread",
    "rating": null
  }
]
```

既存テストが引き続き通ることも確認する。

```bash
npm run ci
# tsc --noEmit && vitest run && eslint
```

Prismaを追加してもService層のテストは一切変更不要だ。
Mock経由でDBから切り離されているため、依存が増えても13件のテストはそのまま通る。

---

## 所感：「テストしない層」を意図的に決める

今回 `PrismaBookRepository` のテストを書かなかった。
これは手を抜いたわけではなく、**意図的な設計判断**だ。

Part1〜2でテスト可能な層を分離してきた流れを振り返る。

| 層 | テスト | 理由 |
|---|---|---|
| entity / valueobject / policy | ✅ 書く | 副作用なし。純粋関数的に検証できる |
| service | ✅ 書く | MockでDB排除。ビジネスロジックを検証 |
| repository | ❌ 書かない | DBが必要。ロジックを持たせない設計 |
| route handler | ❌ 書かない | フレームワーク統合部分。E2Eで担保 |

Repositoryにテストを書かない理由は「面倒だから」ではない。
**ここにビジネスロジックを書かない設計にした**からだ。

ロジックがなければテストする意味も薄い。
`toDomain` と `toRecord` はデータ変換だけ。
バリデーションはValueObjectが担う。
状態遷移ルールはPolicyが担う。

「どこにロジックを置くか」を先に決めたからこそ、「どこをテストしないか」が自然に決まった。

---

## まとめ

Part3でやったこと：

- Prismaをセットアップし、SQLiteにBookテーブルを作った
- `PrismaBookRepository` でDBレコードとドメインオブジェクトの変換を実装した
- Route HandlerでDIを組み立て、`GET /api/books` と `POST /api/books` を動かした
- 既存13テストは変更なしで通り続けることを確認した

Part1から一貫して「**テストできる設計を先に作る**」という方針で進めてきた。
そのおかげでPart3のインフラ層追加が既存コードへの影響ゼロで済んだ。

次は `PATCH /api/books/:id` でステータス変更と評価を繋ぐ予定。
