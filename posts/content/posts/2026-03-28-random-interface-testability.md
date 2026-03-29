---
title: "乱数を interface で抽象化してゲームロジックをテスタブルにする"
date: 2026-03-28T12:00:00+09:00
tags: ["TypeScript", "テスト", "vitest", "Port/Adapter", "設計"]
categories: ["開発"]
---

# 乱数を interface で抽象化してゲームロジックをテスタブルにする

## 概要

ゲーム開発において、「運」の要素は面白さを生む不可欠なスパイスです。クリティカルヒット、レアアイテムのドロップ、ダンジョンの自動生成など、多くの場面で乱数が使われます。

しかし、プログラミングの文脈において、乱数は「不確実性」そのものであり、ユニットテストの天敵です。本記事では、TypeScript を用いて乱数をインターフェースで抽象化し、テスト時に決定論的な（結果が予測可能な）挙動をさせる設計パターンについて解説します。

---

## 課題：`Math.random()` がもたらす「テスト不能」という病

もっとも素浦に実装すると、ゲームロジックの中で直接 `Math.random()` を呼び出すことになります。

```typescript
// 直接 Math.random() を使う例
export class CombatService {
  calculateDamage(baseDamage: number, critRate: number): number {
    // 運が悪ければテストが落ちる
    if (Math.random() < critRate) {
      return baseDamage * 2;
    }
    return baseDamage;
  }
}
```

このコードをテストしようとすると、以下の問題に直面します。

1.  **非決定的 (Non-deterministic) なテスト**: 同じ入力に対して、実行するたびに結果が変わる可能性があります。
2.  **境界値のテストが困難**: 「クリティカル率 5% のとき、0.049 を引いたらクリティカル、0.051 を引いたら通常攻撃」という境界値の検証が運任せになります。
3.  **モックの乱立**: `vi.spyOn(Math, 'random')` などでグローバルなオブジェクトを書き換える手法もありますが、並列テストで干渉したり、クリーンアップを忘れると他のテストに影響を与えたりと、脆いテストになりがちです。

---

## 設計：Port / Adapter パターンによる抽象化

この問題を解決するために、**Dependency Inversion Principle (依存性逆転の原則)** を適用します。

ロジックが「具体的な乱数生成器」に依存するのではなく、抽象的な「乱数提供インターフェース」に依存するように設計を変更します。

### 1. Port (Interface) の定義
ロジックが必要とする「乱数を得るための窓口」を定義します。

### 2. Adapter (Implementation) の実装
- **Production Adapter**: 本番環境では `Math.random()` を使う。
- **Test Adapter**: テスト環境では、事前に定義した値を返したり、シード値に基づいた再現性のある乱数を返す。

この設計にすることで、ロジック側は「誰がどうやって乱数を作っているか」を気にせず、「乱数がもらえること」だけを約束された状態になります。

---

## 実装：RandomPort と各種 Adapter

実際に TypeScript で実装してみましょう。

### RandomPort インターフェース

まずは、共通の型を定義します。単に 0〜1 の値を返すだけでなく、整数の範囲指定など便利なメソッドも持たせると使い勝手が良くなります。

```typescript
/**
 * 乱数生成の抽象ポート
 */
export interface RandomPort {
  /** 0以上1未満の浮動小数を返す */
  next(): number;
  /** min以上max以下の整数を返す */
  nextInt(min: number, max: number): number;
}
```

### 本番用：MathRandomAdapter

標準の `Math.random()` をラップするだけの実装です。

```typescript
export class MathRandomAdapter implements RandomPort {
  next(): number {
    return Math.random();
  }

  nextInt(min: number, max: number): number {
    return Math.floor(this.next() * (max - min + 1)) + min;
  }
}
```

### テスト用：SeededRandomAdapter

テストのために、**シード値（種）を指定すると必ず同じ順序で数値を返す**アダプターを作成します。ここでは簡易的な線形合同法（LCG）を用いた例を示します。

```typescript
export class SeededRandomAdapter implements RandomPort {
  private seed: number;

  constructor(seed: number = 42) {
    this.seed = seed;
  }

  next(): number {
    // 簡易的な LCG アルゴリズム
    this.seed = (this.seed * 1664525 + 1013904223) % 4294967296;
    return this.seed / 4294967296;
  }

  nextInt(min: number, max: number): number {
    return Math.floor(this.next() * (max - min + 1)) + min;
  }
}
```

---

## 実践：戦闘ロジックとドロップ判定

この `RandomPort` を使って、架空の RPG の戦闘・ドロップロジックを書いてみます。

```typescript
interface Item {
  id: string;
  name: string;
}

export class LootSystem {
  // コンストラクタでインターフェースを注入 (DI)
  constructor(private random: RandomPort) {}

  /**
   * モンスターを倒した時のドロップ判定
   * dropRate: 0.0 ~ 1.0
   */
  tryGetLoot(item: Item, dropRate: number): Item | null {
    if (this.random.next() < dropRate) {
      return item;
    }
    return null;
  }

  /**
   * 複数のアイテムから1つを選択（重み付けなし）
   */
  pickOne<T>(items: T[]): T {
    const index = this.random.nextInt(0, items.length - 1);
    return items[index];
  }
}
```

### なぜこの設計にしたか

この設計の肝は、`LootSystem` が **`Math.random()` というグローバルな副作用から切り離されている**点です。

`LootSystem` のインスタンス化の際に `SeededRandomAdapter` を渡せば、その `LootSystem` は「1回目の `next()` は 0.123、2回目は 0.456...」というように、**何度実行しても、どのマシンで実行しても同じ挙動**をします。

---

## テスト例：Vitest による決定論的テスト

それでは、Vitest を使ってテストを書いてみましょう。

```typescript
import { describe, it, expect } from 'vitest';
import { LootSystem } from './LootSystem';
import { SeededRandomAdapter } from './SeededRandomAdapter';

describe('LootSystem', () => {
  const mockItem = { id: 'potion', name: 'ポーション' };

  it('ドロップ率100%なら必ずアイテムが手に入ること', () => {
    // このテストでは乱数の質は関係ないので、何でも良い
    const lootSystem = new LootSystem(new SeededRandomAdapter());
    const result = lootSystem.tryGetLoot(mockItem, 1.0);
    expect(result).toEqual(mockItem);
  });

  it('シード値を固定することで、特定のドロップ結果を再現できること', () => {
    // シード 12345 において、最初の next() が 0.5 以上になることを知っているとする
    // (あるいは、境界値を攻めるために固定値アダプターを作っても良い)
    const seed = 12345;
    const rng = new SeededRandomAdapter(seed);
    const lootSystem = new LootSystem(rng);

    // ドロップ率が非常に低い場合
    const result = lootSystem.tryGetLoot(mockItem, 0.00001);
    
    // シード固定により、この結果は常に null になる（決定論的）
    expect(result).toBeNull();
  });

  it('pickOne が配列の範囲内でランダムに選択すること', () => {
    const rng = new SeededRandomAdapter(999);
    const lootSystem = new LootSystem(rng);
    const items = ['A', 'B', 'C', 'D'];

    // 100回試行しても、必ず範囲内のものが選ばれ、かつ特定のシードなら順序も固定
    for (let i = 0; i < 100; i++) {
      const picked = lootSystem.pickOne(items);
      expect(items).toContain(picked);
    }
  });
});
```

さらに、もっと厳密に「特定の値を返してほしい」場合は、以下のようなシンプルな `Stub` を作ることも可能です。

```typescript
class StubRandomAdapter implements RandomPort {
  constructor(public value: number) {}
  next() { return this.value; }
  nextInt() { return Math.floor(this.value); }
}

it('境界値テスト：ドロップ率 0.05 のとき 0.049 ならドロップする', () => {
  const rng = new StubRandomAdapter(0.049);
  const lootSystem = new LootSystem(rng);
  expect(lootSystem.tryGetLoot(mockItem, 0.05)).not.toBeNull();
});
```

---

## まとめ

乱数を interface で抽象化することには、単に「テストができるようになる」以上のメリットがあります。

1.  **テストの信頼性**: 運に左右される「不安定なテスト (Flaky Tests)」を排除できます。
2.  **デバッグの容易性**: バグが発生した時のシード値をログに残しておけば、開発環境でそのシード値を使って全く同じ状況を再現できます。
3.  **リプレイ機能の実現**: ゲームの対戦ログなどを保存する際、すべての乱数結果を保存する代わりに、初期シード値だけを保存すれば、後から全く同じ展開を再現できます。
4.  **コードの意図の明確化**: コンストラクタで `RandomPort` を要求することで、「このクラスは内部でランダムな挙動を含む」ということを型レベルで明示できます。

「外部への依存（この場合は実行環境の乱数生成器）」をインターフェースの境界線の外側に押し出す。このシンプルな原則を守るだけで、あなたのゲームロジックは格段に堅牢で、メンテナンスしやすいものになるはずです。
