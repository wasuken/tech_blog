---
title: "TypeScript npm workspaces でゲームロジックを UI から完全分離する"
date: 2026-03-28T11:00:00+09:00
tags: ["TypeScript", "monorepo", "設計", "アーキテクチャ", "Port/Adapter"]
categories: ["開発"]
---

# TypeScript npm workspaces でゲームロジックを UI から完全分離する

## 概要

モダンなフロントエンド開発、特にゲーム開発において、「ロジック」と「表示（UI）」の分離は永遠の課題です。React や Vue などのフレームワークにロジックが密結合してしまうと、テストが困難になり、将来的に別のプラットフォーム（例えば Web から React Native や CLI ツールへ）に展開する際の大きな障害となります。

本記事では、**TypeScript npm workspaces** を活用して、ゲームロジックを独立したパッケージ (`packages/core`) として切り出し、React UI (`apps/client`) から完全に分離する設計手法を解説します。また、外部 I/O や非決定的な処理（乱数など）を抽象化する **Port / Adapter パターン**についても触れます。

---

## 課題：なぜロジックが UI に染み出すのか？

多くのプロジェクトでは、気づかないうちにロジックが React コンポーネントや Hooks の中に漏れ出していきます。

```tsx
// 密結合な例
const PlayerStats = () => {
  const [hp, setHp] = useState(100);

  const handleAttack = () => {
    // UI の中で計算ロジックが動いている
    const damage = Math.floor(Math.random() * 10) + 5;
    setHp(prev => Math.max(0, prev - damage));
  };

  return <button onClick={handleAttack}>攻撃を受ける</button>;
};
```

このような設計には以下の課題があります：

1.  **テストの困難さ**: `Math.random()` が直接使われているため、結果が不安定でユニットテストが書きにくい。
2.  **再利用性の欠如**: この「ダメージ計算ロジック」を、サーバーサイドや別の UI フレームワークで使い回すことができない。
3.  **依存の混入**: ロジックを動かすために React の実行環境（レンダリングサイクル）が必要になる。

---

## 設計：npm workspaces による物理的隔離

ロジックを「物理的に」隔離するために、以下の monorepo 構成を採用します。

### ディレクトリ構成

```text
.
├── package.json          # 全体管理
├── packages/
│   └── core/             # 純粋なゲームロジック（React 依存ゼロ）
│       ├── package.json
│       ├── src/
│       │   ├── domain/   # 状態定義・Reducer
│       │   └── port/     # I/O 抽象化（Interface）
└── apps/
    └── client/           # React アプリケーション
        ├── package.json
        └── src/
            ├── adapters/ # I/O 実装（Class）
            └── hooks/    # core を React で使うためのブリッジ
```

### なぜこの設計にするのか

-   **依存の一方向化**: `apps/client` は `packages/core` に依存しますが、その逆は決して許されません。
-   **副作用の制御**: 乱数生成や保存処理などを `Port`（インターフェース）として定義し、実装を `Adapter` として外部から注入することで、ロジックを純粋関数に保ちます。

---

## 実装：ロジックの分離と I/O の抽象化

それでは、具体的な実装を見ていきましょう。

### 1. ワークスペースの設定

ルートの `package.json` で workspaces を宣言します。

```json
{
  "name": "my-game-project",
  "private": true,
  "workspaces": [
    "packages/*",
    "apps/*"
  ]
}
```

### 2. packages/core でのロジック実装

`core` パッケージでは、React に依存せず、TypeScript の型と純粋関数のみでゲームを表現します。

#### Port（インターフェース）の定義
乱数生成など、環境に依存する処理を抽象化します。

```typescript
// packages/core/src/port/random-port.ts
export interface RandomPort {
  nextInt(min: number, max: number): number;
  next(): number;
}
```

#### ドメインロジックの定義
ゲームの状態遷移を Reducer パターンで実装します。

```typescript
// packages/core/src/domain/game-reducer.ts
import { RandomPort } from '../port/random-port';

export type GameState = {
  hp: number;
  status: 'alive' | 'dead';
};

export type GameAction = { type: 'TAKE_DAMAGE'; amount: number };

export function gameReducer(
  state: GameState,
  action: GameAction,
  random: RandomPort // 外部から注入
): GameState {
  switch (action.type) {
    case 'TAKE_DAMAGE':
      const actualDamage = action.amount + random.nextInt(0, 5); // 乱数を使用
      const newHp = Math.max(0, state.hp - actualDamage);
      return {
        ...state,
        hp: newHp,
        status: newHp <= 0 ? 'dead' : 'alive',
      };
    default:
      return state;
  }
}
```

### 3. apps/client での実装

UI 側では、`core` で定義された `Port` の具体的な実装（Adapter）を用意し、ロジックを呼び出します。

#### Adapter（実装）の作成

```typescript
// apps/client/src/adapters/math-random-adapter.ts
import { RandomPort } from '@my-game/core';

export class MathRandomAdapter implements RandomPort {
  nextInt(min: number, max: number): number {
    return Math.floor(Math.random() * (max - min + 1)) + min;
  }
  next(): number {
    return Math.random();
  }
}
```

#### React との接続

`useReducer` を使って、UI とロジックを繋ぎます。

```tsx
// apps/client/src/hooks/useGame.ts
import { useReducer } from 'react';
import { gameReducer, GameState } from '@my-game/core';
import { MathRandomAdapter } from '../adapters/math-random-adapter';

const random = new MathRandomAdapter();

export const useGame = (initialState: GameState) => {
  // core の reducer を React の dispatch に変換する
  const [state, dispatch] = useReducer(
    (s, a) => gameReducer(s, a, random), 
    initialState
  );

  return { state, dispatch };
};
```

---

## この設計がもたらすメリット

### 1. 決定論的なテストが可能になる

`RandomPort` をモックに差し替えることで、常に同じ結果が返るテストが書けます。

```typescript
// packages/core/tests/game.test.ts
const mockRandom = { nextInt: () => 0, next: () => 0 }; // 常に 0 を返す
const nextState = gameReducer({ hp: 100, status: 'alive' }, { type: 'TAKE_DAMAGE', amount: 10 }, mockRandom);
expect(nextState.hp).toBe(90); // 10 + 0 = 10 ダメージ
```

### 2. UI ライブラリのアップデートに強い

もし将来的に React から別のフレームワークに乗り換えることになっても、`packages/core` は一切変更する必要がありません。`apps/new-client` を作るだけで済みます。

### 3. 開発効率の向上

UI を作らなくても、`core` パッケージだけでゲームのルールが正しいかをテストコードで検証できます。これは、複雑な RPG やシミュレーションゲームにおいて非常に強力な武器になります。

---

## まとめ

TypeScript npm workspaces を使ったパッケージの分離は、一見すると初期設定の手間がかかるように見えます。しかし、**「ロジックは純粋であるべき」「環境への依存は外部から注入する」**という原則を物理的に強制することで、中長期的なメンテナンスコストは劇的に下がります。

大規模なゲーム開発はもちろん、小規模なプロジェクトでも「将来の自分への投資」として、ぜひこの Port / Adapter パターンを検討してみてください。
