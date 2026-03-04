---
title: "テスト前提で設計したWebアプリのハンズオン - 読書管理アプリ その2"
date: 2026-03-04T02:00:00+09:00
draft: false
tags: ["TypeScript", "テスト", "設計", "vitest", "Next.js", "クリーンアーキテクチャ", "DI"]
categories: ["開発"]
---

## 前回のおさらい

[その1](/posts/readmeter-part1)では ValueObject・Policy・Entity を作った。ポイントは「フレームワークを一切使わないので、テストがただの関数呼び出しになる」という体験だった。

今回はいよいよ Service 層に入る。**ここが設計の核心だ。** 複数のクラスをまたぐフローを、DI（依存性の注入）を使って DB から切り離す。

---

## 今回追加したもの

```
src/
  domain/
    entity/
      User.ts              # 追加
      Review.ts            # 追加
    valueobject/
      UserName.ts          # 追加
      ReviewComment.ts     # 追加
  repository/
    BookRepository.ts      # 追加（Interface）
  service/
    BookShelfService.ts    # 追加
```

---

## User と Review を追加する

まず Entity の準備。今回は単純なので ValueObject から作る。

### UserName

```typescript
export class UserName {
  constructor(public readonly value: string) {
    if (!value || value.trim().length === 0) {
      throw new Error("UserNameは空にできない");
    }
    if (value.length > 50) {
      throw new Error("UserNameは50文字以内");
    }
  }
}
```

`trim()` してから空チェックしているのがポイント。スペースだけのユーザー名を弾く。

### ReviewComment

```typescript
export class ReviewComment {
  constructor(public readonly value: string) {
    if (!value || value.trim().length === 0) {
      throw new Error("ReviewCommentは空にできない");
    }
    if (value.length > 1000) {
      throw new Error("ReviewCommentは1000文字以内");
    }
  }
}
```

### User Entity

```typescript
import { UserName } from "../valueobject/UserName";

export class User {
  constructor(
    public readonly id: string,
    public readonly name: UserName,
  ) {}
}
```

`User` 自体はシンプルだ。ロジックがない。ビジネスルールが増えれば `changeXxx()` メソッドが生える設計だが、今は id と name を持つだけでいい。

### Review Entity

```typescript
import { Rating } from "../valueobject/Rating";
import { ReviewComment } from "../valueobject/ReviewComment";

export class Review {
  constructor(
    public readonly id: string,
    public readonly bookId: string,
    public readonly userId: string,
    public readonly rating: Rating,
    public readonly comment: ReviewComment,
  ) {}
}
```

`Review` は `Book` と `User` を **ID だけで参照している**。オブジェクト参照を持たない。こうすることで `Review` 単体をテストするときに `Book` や `User` の実態を用意しなくていい。

---

## BookRepository Interface を定義する

```typescript
import { Book } from "../domain/entity/Book";

export interface BookRepository {
  save(book: Book): Promise<void>;
  findById(id: string): Promise<Book | null>;
  findAll(): Promise<Book[]>;
  delete(id: string): Promise<void>;
}
```

**これは実装ではなく契約だ。** DB が SQLite でも PostgreSQL でも、この Interface を満たせば差し替えられる。

実際のDB実装（`PrismaBookRepository` など）は `repository/` 層に置く。でも今はまだ作らない。Service とそのテストを書くのに、DB 実装は一切不要なのがこの設計のメリット。

---

## BookShelfService を実装する

```typescript
import { Book } from "../domain/entity/Book";
import { ISBN } from "../domain/valueobject/ISBN";
import { Rating } from "../domain/valueobject/Rating";
import { ReadingStatus } from "../domain/valueobject/ReadingStatus";
import { BookRepository } from "../repository/BookRepository";

export class BookShelfService {
  constructor(private readonly repo: BookRepository) {}

  async addBook(id: string, title: string, isbnValue: string): Promise<Book> {
    const isbn = new ISBN(isbnValue);
    const book = new Book(id, title, isbn, ReadingStatus.Unread, null);
    await this.repo.save(book);
    return book;
  }

  async startReading(id: string): Promise<Book> {
    const book = await this.repo.findById(id);
    if (!book) throw new Error(`Book not found: ${id}`);
    book.changeStatus(ReadingStatus.Reading);
    await this.repo.save(book);
    return book;
  }

  async completeReading(id: string, ratingValue?: number): Promise<Book> {
    const book = await this.repo.findById(id);
    if (!book) throw new Error(`Book not found: ${id}`);
    book.changeStatus(ReadingStatus.Completed);
    if (ratingValue !== undefined) {
      book.addRating(new Rating(ratingValue));
    }
    await this.repo.save(book);
    return book;
  }

  async getAll(): Promise<Book[]> {
    return this.repo.findAll();
  }

  async removeBook(id: string): Promise<void> {
    const book = await this.repo.findById(id);
    if (!book) throw new Error(`Book not found: ${id}`);
    await this.repo.delete(id);
  }
}
```

コンストラクタで `BookRepository` を受け取っている。**Service は Interface しか知らない。** 実装クラスの名前すら import していない。

状態遷移のロジック自体は `Book.changeStatus()` に委譲している。Service は「どの順番で何を呼ぶか」のフロー制御だけを担う。

---

## Mock を使って Service をテストする

ここが今回の肝。**本物の DB をまったく使わずに Service をテストする。**

```typescript
function createMockRepo(): BookRepository {
  const store = new Map<string, Book>();
  return {
    save: async (book) => { store.set(book.id, book); },
    findById: async (id) => store.get(id) ?? null,
    findAll: async () => Array.from(store.values()),
    delete: async (id) => { store.delete(id); },
  };
}
```

`Map` を使ったインメモリ実装。これが Mock だ。DB のスキーマも Prisma の設定も不要。**Interface の契約を満たしているだけ。**

意図的にクラスにしていない。テストファイルの中だけに存在するべきで、外に漏れる必要がないからだ。

テストはこうなる。

```typescript
test("積読→読中に変更できる", async () => {
  const service = new BookShelfService(createMockRepo());
  await service.addBook("1", "Clean Code", "9784048860000");
  const book = await service.startReading("1");

  expect(book.status).toBe(ReadingStatus.Reading);
});

test("積読から直接読了はエラー", async () => {
  const service = new BookShelfService(createMockRepo());
  await service.addBook("1", "Clean Code", "9784048860000");

  await expect(service.completeReading("1")).rejects.toThrow(
    "UnreadからCompletedへの変更は不可",
  );
});
```

**DB なし、Next.js なし、FW なし。** 相変わらずただのクラスと関数呼び出しだ。非同期なので `await` が入っているが、それだけ。

全9テストケースを書いた。

| テストケース | 内容 |
|---|---|
| 本を追加できる | 初期状態は Unread |
| 積読→読中 | startReading で状態変更 |
| 読中→読了（評価あり） | completeReading で rating がセット |
| 読中→読了（評価なし） | rating は null のまま |
| 積読→読了はエラー | Policy による遷移制限 |
| 存在しない本の startReading | not found エラー |
| 全件取得 | 2冊追加して length 確認 |
| 本を削除できる | 削除後に length 0 |
| 存在しない本の削除 | not found エラー |

---

## 実行結果

```
✓ src/service/BookShelfService.test.ts (9 tests)
✓ src/domain/valueobject/ReviewComment.test.ts (1 test)
✓ src/domain/entity/Review.test.ts (1 test)
✓ src/domain/valueobject/UserName.test.ts (1 test)
✓ src/domain/entity/User.test.ts (1 test)

Test Files  5 passed (5)
     Tests  13 passed (13)
```

Part1 からの累計で 13 テスト全パス。

---

## DI を整理する

今回やったことを図にすると以下だ。

```
テスト時:
  BookShelfService ← MockBookRepository（Map）

本番時（将来）:
  BookShelfService ← PrismaBookRepository（SQLite）
```

Service のコードは一行も変わらない。`BookRepository` という Interface だけを見ているから、差し込むものを変えればいい。

CI4 の Model との違いを言語化するとこうだ。

| | CI4 Model | Repository Interface |
|---|---|---|
| DB を知っているのは | Model 自身 | Repository 実装クラスのみ |
| テスト時の切り離し | 難しい（ActiveRecord） | Interface ごと差し替える |
| Service から見た DB | 見えてしまう | Interface の向こう側 |

**テストが書きにくい、というのは設計の問題だ。** 今回の構造にしておけば、DB を一切用意しなくてもビジネスロジックの全パスをテストできる。

---

## 所感

正直、ここまでは「なんとなく理解していた」が「体で動かしたことはなかった」レベルだった。

実際に `createMockRepo()` を書いて Service に差し込んだとき、「あ、これで DB 要らないじゃん」という実感が来た。理屈では知っていたが、手を動かして初めて腹落ちした。

Mock を別ファイルに切り出さなかった理由も、実装しながら考えた結果だ。テスト専用の実装を本番コードのディレクトリに置いてしまうと、境界が曖昧になる。テストファイルの中だけで完結させておく方が「これはテスト用だ」という意図が明確になる。

次回は Prisma 導入と実際の DB 実装に入る。Repository Interface の実装クラスをついに作る。テストは書かないが、この層を作ることで「本物のアプリ」になる。

---

## 次やること（Part3）

1. Prisma セットアップ（SQLite）
2. `PrismaBookRepository` 実装
3. Next.js の Route Handler で API エンドポイント作成
4. 動作確認
