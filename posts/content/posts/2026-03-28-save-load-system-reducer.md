---
title: "React で作る堅牢なゲームセーブシステム：localStorage と Reducer を疎結合に保つ設計"
date: 2026-03-28T15:00:00+09:00
tags: ["React", "localStorage", "Reducer", "設計", "TypeScript"]
categories: ["開発"]
---

# React で作る堅牢なゲームセーブシステム：localStorage と Reducer を疎結合に保つ設計

ゲーム開発において、プレイヤーの進行状況を保存する「セーブ / ロード」は最も重要な機能の一つです。Web ブラウザで動作する React アプリケーションの場合、最も手軽な保存先は `localStorage` です。

しかし、単純に `localStorage.setItem` をコードのあちこちに散りばめてしまうと、テスタビリティが低下し、データ構造の変更（スキーマ変更）に弱いシステムになってしまいます。

本記事では、架空の RPG 『React Odyssey』を例に、`SavePort` インターフェースと `Reducer` の初期化関数を組み合わせた、クリーンで堅牢なセーブ / ロードの実装方法を解説します。

---

## 1. 概要：なぜ「直接 localStorage」を避けるのか

React の `useReducer` を使ったゲーム開発では、ゲームの状態（State）は一つの大きなオブジェクトとして管理されます。これを `JSON.stringify` して `localStorage` に保存するのは簡単です。

しかし、以下の理由から、ロジックの中に直接 `localStorage` を書くべきではありません。

1.  **副作用の分離**: Reducer は純粋関数であるべきです。セーブ処理（副作用）を Reducer の中に入れることはできません。
2.  **環境非依存**: 将来的に保存先を `IndexedDB` やクラウド（Firebase 等）に変更したくなったとき、コードを大幅に書き換える必要が出てきます。
3.  **テストのしやすさ**: `localStorage` が存在しない Node.js 環境（Vitest 等）でロジックのテストを行う際、モック化が容易である必要があります。

これらを解決するために、**Dependency Inversion Principle（依存性逆転の原則）** に基づいた設計を採用します。

---

## 2. SavePort 設計：抽象化の定義

まずは、ゲームエンジン側が必要とする「セーブ機能」のインターフェースを定義します。これを `SavePort` と呼びます。

```typescript
// domain/save/save-port.ts

/** 
 * 保存されるデータの構造（スキーマ）
 * ゲームの現在の状態に加え、メタ情報を付与する
 */
export interface SaveData {
  version: number;        // セーブデータのバージョン
  timestamp: string;      // 保存日時
  state: GameState;       // 実際のゲーム状態
}

/**
 * セーブ/ロードに関する抽象インターフェース
 */
export interface SavePort {
  /** データを保存する */
  save(data: SaveData): Promise<void>;

  /** データを読み込む。存在しない場合は null を返す */
  load(): Promise<SaveData | null>;

  /** セーブデータが存在するか確認する */
  exists(): Promise<boolean>;

  /** セーブデータを削除する */
  clear(): Promise<void>;
}
```

**なぜインターフェースにするのか？**
ゲームのメインロジック（UseCase）は、この `SavePort` を通じて読み書きを行います。この時点では「どうやって保存するか」は知らなくてよく、「保存できる」という事実だけに依存させます。

---

## 3. LocalStorageSaveAdapter 実装：具体化の責務

次に、`SavePort` を `localStorage` を使って具体的に実装した「アダプター」を作成します。

```typescript
// adapters/localstorage-save-adapter.ts
import { SavePort, SaveData } from '../domain/save/save-port';

export class LocalStorageSaveAdapter implements SavePort {
  private readonly STORAGE_KEY = 'react_odyssey_save_v1';

  async save(data: SaveData): Promise<void> {
    try {
      const serialized = JSON.stringify(data);
      localStorage.setItem(this.STORAGE_KEY, serialized);
    } catch (error) {
      console.error('Failed to save data to localStorage:', error);
      throw new Error('セーブに失敗しました。ディスク容量を確認してください。');
    }
  }

  async load(): Promise<SaveData | null> {
    const serialized = localStorage.getItem(this.STORAGE_KEY);
    if (!serialized) return null;

    try {
      return JSON.parse(serialized) as SaveData;
    } catch (error) {
      console.error('Failed to parse save data:', error);
      return null; // 破損している場合は null を返して初期状態にする
    }
  }

  async exists(): Promise<boolean> {
    return localStorage.getItem(this.STORAGE_KEY) !== null;
  }

  async clear(): Promise<void> {
    localStorage.removeItem(this.STORAGE_KEY);
  }
}
```

**設計のポイント：**
- `STORAGE_KEY` を一箇所に定義し、他から参照させないことで、キーの重複やミスを防ぎます。
- `try-catch` をアダプター内で完結させ、呼び出し側（React コンポーネント）にはクリーンな結果（または意味のあるエラーメッセージ）を返します。

---

## 4. Reducer 初期値へのロード：非同期と初期化の壁

React の `useReducer` にロードしたデータを反映させるには、少し工夫が必要です。`useReducer` の第3引数である `initializer` 関数を利用します。

まずは、ゲームの状態と Reducer の定義を見てみましょう。

```typescript
// domain/game/game-reducer.ts

export interface GameState {
  player: { hp: number; mp: number; gold: number };
  location: string;
}

export const INITIAL_STATE: GameState = {
  player: { hp: 100, mp: 50, gold: 0 },
  location: '始まりの町'
};

export type GameAction = 
  | { type: 'RECOVER_HP'; amount: number }
  | { type: 'MOVE'; to: string }
  | { type: 'LOAD_GAME'; state: GameState }; // ロード専用のアクション

export function gameReducer(state: GameState, action: GameAction): GameState {
  switch (action.type) {
    case 'RECOVER_HP':
      return { ...state, player: { ...state.player, hp: state.player.hp + action.amount } };
    case 'MOVE':
      return { ...state, location: action.to };
    case 'LOAD_GAME':
      return action.state; // 保存された状態で上書き
    default:
      return state;
  }
}
```

### コンポーネントでのロード処理

`localStorage.getItem` は同期処理ですが、将来の `IndexedDB` 移行を見据えて `async` に対応させる必要があります。React の `useEffect` でロードを実行し、成功したらアクションをディスパッチします。

```tsx
// components/GameProvider.tsx
import React, { useReducer, useEffect, useMemo } from 'react';
import { gameReducer, INITIAL_STATE, GameState } from '../domain/game/game-reducer';
import { LocalStorageSaveAdapter } from '../adapters/localstorage-save-adapter';

export const GameContext = React.createContext<{
  state: GameState;
  dispatch: React.Dispatch<any>;
  saveGame: () => void;
} | null>(null);

export const GameProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [state, dispatch] = useReducer(gameReducer, INITIAL_STATE);
  
  // アダプターのインスタンスをメモ化
  const saveAdapter = useMemo(() => new LocalStorageSaveAdapter(), []);

  // アプリ起動時にロードを試みる
  useEffect(() => {
    const initLoad = async () => {
      const savedData = await saveAdapter.load();
      if (savedData) {
        dispatch({ type: 'LOAD_GAME', state: savedData.state });
      }
    };
    initLoad();
  }, [saveAdapter]);

  // 手動セーブ用の関数
  const saveGame = async () => {
    const data = {
      version: 1,
      timestamp: new Date().toISOString(),
      state: state
    };
    await saveAdapter.save(data);
    alert('セーブが完了しました！');
  };

  return (
    <GameContext.Provider value={{ state, dispatch, saveGame }}>
      {children}
    </GameContext.Provider>
  );
};
```

---

## 5. セーブデータの破損・バージョン不一致への対策

ここまでの実装では、データ構造が変わったときにクラッシュする危険があります。例えば、`player` オブジェクトに `level` フィールドを追加した後に古いセーブデータを読み込むと、`level` が `undefined` になり、計算で `NaN` が発生するかもしれません。

### 対策1：バージョンチェックと移行（Migration）

`SaveData` に含めた `version` をチェックします。

```typescript
function migrate(data: any): GameData {
  let currentData = data;

  // v1 から v2 への移行
  if (currentData.version === 1) {
    currentData = {
      ...currentData,
      version: 2,
      state: {
        ...currentData.state,
        player: { ...currentData.state.player, level: 1 } // 新しいフィールドを追加
      }
    };
  }

  return currentData;
}
```

### 対策2：スキーマバリデーションとフォールバック

ロードしたデータが正しい形式か、実行時にチェックします。簡易的な方法としては、スプレッド構文を用いた「デフォルト値の流し込み」が有効です。

```typescript
// アダプターの load 内で、構造の不一致を最小限に抑える
async load(): Promise<SaveData | null> {
  const serialized = localStorage.getItem(this.STORAGE_KEY);
  if (!serialized) return null;

  try {
    const rawData = JSON.parse(serialized);
    
    // 最小限の構造チェック
    if (!rawData.state || !rawData.version) {
      throw new Error('Invalid save data format');
    }

    // デフォルト値をマージして、新しく追加されたフィールドの欠落を防ぐ
    const sanitizedState = {
      ...INITIAL_STATE,
      ...rawData.state,
      player: {
        ...INITIAL_STATE.player,
        ...rawData.state.player
      }
    };

    return { ...rawData, state: sanitizedState };
  } catch (error) {
    console.warn('Save data is corrupted. Starting a new game.', error);
    return null;
  }
}
```

---

## 6. まとめ

本記事では、React でゲームのセーブ / ロードを実装する際の「疎結合」な設計について解説しました。

1.  **SavePort (Interface)** で「何をすべきか」を定義する。
2.  **LocalStorageSaveAdapter (Class)** で「どう保存するか」を実装する。
3.  **Reducer** は副作用を持たず、専用のアクション（`LOAD_GAME`）で状態を受け取る。
4.  **サニタイズ処理** を挟むことで、コードの進化によるデータの破損から守る。

この設計の最大の利点は、**テストコードが書きやすくなること**です。テスト時には `LocalStorageSaveAdapter` の代わりに `MemorySaveAdapter` を渡すだけで、ブラウザ環境なしでセーブ機能の検証が可能になります。

```typescript
// テスト用のモックアダプター
export class MemorySaveAdapter implements SavePort {
  private data: SaveData | null = null;
  async save(d: SaveData) { this.data = d; }
  async load() { return this.data; }
  // ...
}
```

ゲームが複雑になるにつれ、保存すべきデータの量は増えていきます。初期の段階で「保存の責務」を明確に分けておくことで、将来の機能追加やプラットフォーム変更に強いコードベースを維持できるでしょう。

あなたの React Odyssey に、安全な「冒険の記録」を実装してみてください。
