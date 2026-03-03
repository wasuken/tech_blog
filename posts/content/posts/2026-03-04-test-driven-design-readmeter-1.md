---
title: "テスト前提で設計したWebアプリのハンズオン - 読書管理アプリ その1"
date: 2026-03-04T01:00:00+09:00
draft: false
tags: ["TypeScript", "テスト", "設計", "vitest", "Next.js", "クリーンアーキテクチャ"]
categories: ["開発"]
---

## なぜこの記事を書いたか

動くものをまず作って、そこにテストを後付けしようとした。しかし、できなかった。

具体的にはこういう壁にぶつかった。

- ModelなどのDB接続の切り替えがうまくいかない
- Controllerをテストしようにもいろいろおかしくなる

悩んだ結果、気づいたことがある。**そもそもそんなものは単体テストじゃない。** ロジックをControllerやModelに全部書いていたのが悪かった。

「じゃあ最初からそう書けばいいじゃないか」という話だが、それが難しい。雑魚プログラマにいきなりクリーンな設計はできない。

なので発想を逆にした。**テスト前提でコードを書く。** テストが書けない場所にはロジックを書かない。それを体で覚えるために、今回のハンズオンを始めた。

---

## 方針

生成AIに実装ヒントと次のステップを教えてもらいながら、コードは自分で書く。

正直、コード自体はAIに書かせてもいいと思っている。ただ、設計の考え方は頭に入れる必要があるので、写経しながら理解を深めている。

**テストする場所の原則はシンプルだ。**

- フレームワークで用意された便利クラスはテストしない
- DBやAPIなど外界と接する場所はテストしない
- 自分で書いた純粋なTS部分だけをテストする

---

## 作るもの

読書メーターのパクり。Todoアプリはビジネスロジックがほぼ存在しないのでテストの練習に向いていない。読書メーターなら本の状態遷移（積読→読中→読了）などのルールが自然に生まれるので、テストの旨味がある。

**技術スタック**

- Next.js
- SQLite + Prisma
- vitest

---

## ディレクトリ構成

```
src/
  domain/
    entity/
    valueobject/
    policy/
  service/
  repository/
```

この構成の考え方は以下の通り。

| 層 | 役割 |
|---|---|
| entity | 概念そのもの（Book, User） |
| valueobject | 値の制約を持つクラス（ISBN, Rating） |
| policy | ビジネスルールを切り出したクラス |
| service | 複数クラスをまたぐフロー |
| repository | DBとのアダプタ（副作用をここに閉じ込める） |

**repositoryはCI4でいうModelに近い立ち位置だ。** ただし決定的な違いがある。CI4のModelはActiveRecordパターンでクラス自身がDBを知っているため切り離せないが、repositoryはInterfaceと実装を分けることで差し替え可能にする。これがDI（依存性の注入）の肝で、テスト時にMockに差し替えられる。

---

## vitestのセットアップ

```bash
npm install -D vitest @vitejs/plugin-react vite-tsconfig-paths
npm install -D @testing-library/react @testing-library/jest-dom
```

`vitest.config.ts`

```typescript
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'
import tsconfigPaths from 'vite-tsconfig-paths'

export default defineConfig({
    plugins: [react(), tsconfigPaths()],
    test: {
        environment: 'jsdom',
        globals: true,
        setupFiles: ['./src/test/setup.ts'],
    },
})
```

`package.json`のscriptsに追加。

```json
"scripts": {
    "test": "vitest",
    "typecheck": "tsc --noEmit",
    "ci": "tsc --noEmit && vitest run"
}
```

`ci`スクリプトがポイントで、vitestは型チェックをしない。`tsc --noEmit`と組み合わせることで、型エラーも含めて検証できる。

---

## 実装したもの

### ReadingStatus

```typescript
export const ReadingStatus = {
    Unread: 'Unread',       // 積読
    Reading: 'Reading',     // 読中
    Completed: 'Completed', // 読了
} as const

export type ReadingStatus = typeof ReadingStatus[keyof typeof ReadingStatus]
```

### ReadingStatusPolicy

状態遷移のルールをここに閉じ込める。

```typescript
import { ReadingStatus } from '../valueobject/ReadingStatus'

export class ReadingStatusPolicy {
    static canTransition(from: ReadingStatus, to: ReadingStatus): boolean {
        const rules: Record<ReadingStatus, ReadingStatus[]> = {
            [ReadingStatus.Unread]: [ReadingStatus.Reading],
            [ReadingStatus.Reading]: [ReadingStatus.Completed],
            [ReadingStatus.Completed]: [ReadingStatus.Reading],
        }
        return rules[from].includes(to)
    }
}
```

テストはこうなる。

```typescript
test("ReadingStatusPolicy.canTransitionの組み合わせチェック", () => {
    expect(ReadingStatusPolicy.canTransition(ReadingStatus.Unread, ReadingStatus.Reading)).toBe(true)
    expect(ReadingStatusPolicy.canTransition(ReadingStatus.Unread, ReadingStatus.Completed)).toBe(false)
    expect(ReadingStatusPolicy.canTransition(ReadingStatus.Reading, ReadingStatus.Completed)).toBe(true)
    expect(ReadingStatusPolicy.canTransition(ReadingStatus.Reading, ReadingStatus.Unread)).toBe(false)
    expect(ReadingStatusPolicy.canTransition(ReadingStatus.Completed, ReadingStatus.Reading)).toBe(true)
    expect(ReadingStatusPolicy.canTransition(ReadingStatus.Completed, ReadingStatus.Unread)).toBe(false)
})
```

**DB一切なし、Next.js一切なし、FW一切なし。** ただのクラスとロジックだけなのでテストが普通に書ける。

### Rating

```typescript
export class Rating {
    constructor(public readonly value: number) {
        if (!Number.isInteger(value) || value < 1 || value > 5) {
            throw new Error('Ratingは1〜5の整数')
        }
    }
}
```

例外テストは`expect(() => ...).toThrow()`で一行で書ける。

```typescript
test('Ratingは1〜5の整数のみ有効', () => {
    expect(() => new Rating(1)).not.toThrow()
    expect(() => new Rating(5)).not.toThrow()
    expect(() => new Rating(0)).toThrow('Ratingは1〜5の整数')
    expect(() => new Rating(6)).toThrow('Ratingは1〜5の整数')
    expect(() => new Rating(3.5)).toThrow('Ratingは1〜5の整数')
})
```

### ISBN

今回は13桁の数字文字列という簡易バリデーションのみ。本来はチェックディジットの検証が必要だが、テストの練習が目的なので一旦これで。

### Book（Entity）

```typescript
export class Book {
    constructor(
        public readonly id: string,
        public readonly title: string,
        public readonly isbn: ISBN,
        public status: ReadingStatus,
        public rating: Rating | null = null,
    ) {}

    changeStatus(to: ReadingStatus): void {
        if (!ReadingStatusPolicy.canTransition(this.status, to)) {
            throw new Error(`${this.status}から${to}への変更は不可`)
        }
        this.status = to
    }

    addRating(rating: Rating): void {
        if (this.status !== ReadingStatus.Completed) {
            throw new Error('読了していない本には評価できない')
        }
        this.rating = rating
    }
}
```

---

## 所感

序盤なので簡単なところだけだが、テストが普通に書けるという体験ができた。

今まで詰まっていたのはやり方が悪かったわけでも実装が悪かったわけでもなく、**フレームワークの設計と戦っていたから**だと気づいた。ロジックをFWから分離してしまえば、テストはただの関数呼び出しになる。

次回はService層の実装でDIが登場する。ここがこの設計の核心部分になるはず。
