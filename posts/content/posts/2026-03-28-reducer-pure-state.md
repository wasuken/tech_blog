---
title: "Reducer パターンだけでゲーム状態を管理する：副作用ゼロのアーキテクチャ"
date: 2026-03-28T14:00:00+09:00
tags: ["Reducer", "アーキテクチャ", "設計", "TypeScript", "テスト"]
categories: ["開発"]
---

# Reducer パターンだけでゲーム状態を管理する：副作用ゼロのアーキテクチャ

## 概要

現代のフロントエンド開発において、Redux や React の `useReducer` を通じて「Reducer パターン」は広く浸透しました。しかし、このパターンを本格的なゲーム開発、特に複雑なロジックが絡み合う RPG やシミュレーションゲームに適用しようとすると、多くの開発者が「副作用」の壁にぶつかります。

「ダメージ計算に乱数を使いたい」「ゲーム内の経過時間を管理したい」「マスターデータを参照したい」……。

これらの要素を素直に実装すると、Reducer の外側にある状態や関数に依存してしまい、純粋関数としての美しさとテストのしやすさが失われてしまいます。

本記事では、すべての副作用を引数として注入し、**`gameReducer(state, action, masters, random)`** という純粋関数のみでゲームの全ロジックを完結させる「副作用ゼロ」のアーキテクチャについて解説します。

---

## 課題：なぜゲーム開発で Reducer は敬遠されるのか

一般的な Web アプリケーションの Reducer は単純です。「ボタンを押したらフラグを反転させる」「入力された文字列を状態に保存する」といった操作がメインだからです。

しかし、ゲームでは以下のような「非決定的な要素」や「巨大な静的データ」が頻繁に登場します。

### 1. 乱数 (Randomness) の扱い
クリティカルヒットの判定、モンスターのドロップアイテム、マップの自動生成など、ゲームは乱数の塊です。Reducer の中で `Math.random()` を呼んだ瞬間、その関数は「同じ入力に対して同じ出力を返す」という純粋性を失います。

### 2. 静的データ (Master Data) の参照
モンスターのステータス表、アイテムの定義、スキル効果など、ゲームには膨大な「変わらないデータ」があります。これを Reducer の外にあるグローバル変数から参照すると、テスト時にそのグローバル変数の状態も気にしなければならなくなります。

### 3. 時刻 (Time) と経過の管理
「3時間経過したらスタミナが回復する」「夜になるとモンスターが強くなる」といった時間依存の処理です。`new Date()` を Reducer で使うのは、乱数と同様に純粋性を破壊します。

### 4. 非同期処理 (Async/Fetch)
サーバーから最新のイベント情報を取得したり、セーブデータをロードしたりする処理です。Reducer は同期的に実行される必要があるため、非同期処理をそのまま中に書くことはできません。

### OOP（オブジェクト指向）との比較
クラスベースの OOP では、`player.attack(monster)` のようにメソッドを呼び出します。これは直感的ですが、内部で `this.hp -= damage` のように状態を直接書き換えます（ミュータブル）。
小規模なら良いですが、プロジェクトが大きくなり「攻撃時にスキルが発動し、その効果で回復し、さらにログに記録し……」と連鎖が始まると、どこで何が起きたかを追跡するのが不可能になります。

---

## 設計：副作用を「引数」に封じ込める

副作用を排除するための解決策はシンプルです。**「必要なものはすべて外から渡す」**、つまり依存性の注入 (DI) です。

目指すべき Reducer のシグネチャは以下の通りです。

```typescript
type GameReducer = (
  state: GameState,     // 現在のゲーム状態（イミュータブル）
  action: GameAction,   // ユーザーの操作やイベント（シリアライズ可能）
  masters: Masters,     // 静的なマスタデータ（読み取り専用）
  random: RandomPort    // 乱数生成器のインターフェース
) => GameState;         // 新しいゲーム状態
```

### なぜこの設計にするのか

-   **完全な再現性:** `state`, `action`, `masters`, `random` のセットが同じなら、結果は 100% 同じになります。バグレポートが届いた際、この4つを再現するだけで、手元で確実にバグを再現できます。
-   **テストの容易性:** 乱数生成器をモック化することで、「1% の確率で発生する超レアドロップ」のテストも 100% の確率で再現させて検証できます。
-   **ロジックの集中:** ゲームのルールがこの関数一つ（とその配下のサブ Reducer）に集約されます。「このルールはどこに書いてある？」と迷うことがなくなります。
-   **プラットフォーム非依存:** Core ロジックが `DOM` や `Node.js` の API に依存しないため、React (Web), React Native (Mobile), Electron (Desktop) で全く同じコードを使い回せます。

---

## 実装：型安全な Reducer の構築

それでは、具体的な実装例を見ていきましょう。TypeScript の Union 型を活用し、`any` を排除した設計にします。

### 1. アクションと状態の定義

まず、ゲーム内で発生するアクションを「データ」として定義します。

```typescript
// アクションの定義：判別可能な Union 型
type GameAction =
  | { type: 'WALK'; direction: 'north' | 'south' | 'east' | 'west' }
  | { type: 'ATTACK_MONSTER'; monsterId: string }
  | { type: 'USE_ITEM'; instanceId: string }
  | { type: 'TICK'; hours: number }
  | { type: 'LOAD_SAVE_DATA'; data: GameState }; // 外部からのロード結果

// 状態の定義
interface GameState {
  player: PlayerState;
  world: WorldState;
  quests: QuestState;
  log: LogEntry[];
  time: GameTime;
}

interface GameTime {
  day: number;
  hour: number;
}
```

### 2. 乱数ポートのインターフェース

`Math.random()` を直接使うのではなく、ラップしたインターフェースを渡します。これにより、テスト時には特定の数値を返す「決定的な乱数生成器」を注入できます。

```typescript
export interface RandomPort {
  next(): number; // 0.0 ~ 1.0
  nextInt(min: number, max: number): number;
}
```

### 3. Reducer の合成 (Composition) と責務分割

一つの Reducer ですべてを書くと巨大になりすぎるため、責務ごとに分割します。これは Redux の `combineReducers` に近い考え方ですが、引数に `masters` や `random` を渡せるように拡張するのがポイントです。

```typescript
export function gameReducer(
  state: GameState,
  action: GameAction,
  masters: Masters,
  random: RandomPort
): GameState {
  // 1. 各サブ Reducer に委譲（各々が純粋関数）
  // プレイヤーの基本ステータス更新などは playerReducer で
  let player = playerReducer(state.player, action, masters);
  
  // マップの状態などは worldReducer で
  let world = worldReducer(state.world, action);
  
  let time = state.time;
  let quests = state.quests;

  // 2. 複数の要素が絡み合う複雑なロジックをトップレベルで記述
  switch (action.type) {
    case 'TICK':
      // 時間を進める（advanceTime も純粋関数）
      time = advanceTime(state.time, action.hours);
      
      // 時間経過によるクエストの自動生成（乱数を使用）
      if (time.hour % 8 === 0) {
        const newQuest = generateRandomQuest(player.rank, time, random, masters);
        quests = { ...quests, available: [...quests.available, newQuest] };
      }
      break;

    case 'ATTACK_MONSTER': {
      const monsterDef = masters.monsters.find(m => m.id === action.monsterId);
      if (!monsterDef) break;

      // 乱数を使用したダメージ計算
      const isCritical = random.next() < (player.stats.luk * 0.01);
      const baseDamage = player.stats.str - monsterDef.def;
      const damage = Math.max(1, isCritical ? baseDamage * 2 : baseDamage);

      // モンスターを倒したか判定（本来は monster の HP も state で持つべきですが簡略化）
      if (damage >= 10) { 
        player = gainExp(player, monsterDef.exp, masters);
      }
      break;
    }
    
    case 'LOAD_SAVE_DATA':
      // 非同期で取得されたデータは、アクションのペイロードとして渡される
      return action.data;
  }

  // 3. 新しい状態を返却
  return {
    ...state,
    player,
    world,
    time,
    quests,
    log: updateLog(state.log, action, state)
  };
}
```

### 4. 非同期処理 (Fetch) の扱い：Result-in-Action パターン

Reducer の外側（React の `useEffect` や Action Creator）で非同期処理を行い、その結果をアクションとして Dispatch します。

```typescript
// React コンポーネント内での例
const handleLoad = async () => {
  const data = await fetchSaveData(); // 外部の副作用
  dispatch({ type: 'LOAD_SAVE_DATA', data }); // 結果だけを Reducer に送る
};
```

これにより、Reducer 自体は常に同期的な純粋関数のままでいられます。

---

## テスト：副作用ゼロがもたらす最強のデバッグ環境

このアーキテクチャの真価はテストで発揮されます。`vitest` や `jest` を使ったテストコードを見てみましょう。

```typescript
import { describe, it, expect } from 'vitest';
import { gameReducer } from './game-reducer';
import { MockRandom } from './test-helpers';

describe('Combat Logic', () => {
  it('should land a critical hit when random value is low', () => {
    // 準備：常に 0.0 を返す乱数生成器（クリティカル確定）
    const mockRandom = new MockRandom(0.0);
    const masters = { /* ... モンスターデータ ... */ };
    const initialState = { /* ... プレイヤーデータ ... */ };

    const action = { type: 'ATTACK_MONSTER', monsterId: 'goblin' };
    
    // 実行
    const newState = gameReducer(initialState, action, masters, mockRandom);

    // 検証：クリティカルダメージが適用されているか
    expect(newState.log[0].message).toContain('クリティカルヒット！');
  });

  it('should increase player exp when a monster is killed', () => {
    const mockRandom = new MockRandom(0.5);
    const initialState = { player: { exp: 0, ... }, ... };
    const action = { type: 'ATTACK_MONSTER', monsterId: 'goblin' };
    
    const newState = gameReducer(initialState, action, masters, mockRandom);
    
    expect(newState.player.exp).toBeGreaterThan(0);
  });
});
```

「100回に1回しか起きないバグ」も、このように `MockRandom` を通じて確実に再現できます。

---

## 運用上の注意とパフォーマンス

### イミュータブル更新のオーバーヘッド
大規模なゲームで毎回オブジェクトをコピー (`{ ...state }`) するのは、メモリや CPU の負荷が懸念されます。
これには **Immer.js** の導入が非常に有効です。Reducer 内部で `draft` を直接書き換えるようなコードが書けますが、最終的な出力はイミュータブルになります。

```typescript
import { produce } from 'immer';

const gameReducer = (state, action, masters, random) => 
  produce(state, draft => {
    switch (action.type) {
      case 'WALK':
        draft.player.x += 1; // 直感的な記述が可能
        break;
    }
  });
```

### 巨大なマスタデータの扱い
マスタデータを毎回引数で渡すのは、一見非効率に見えますが、JavaScript ではオブジェクトは参照渡しされるため、実行時のオーバーヘッドは無視できるほど小さいです。それよりも、どこからでもマスタデータにアクセスできる見通しの良さの方が価値があります。

---

## まとめ

Reducer パターンをゲーム開発に適用し、副作用を徹底的に排除することで、以下のような恩恵が得られます。

1.  **バグの再現が 100% 可能になる:** `state` と `action` の履歴、シード値があれば、宇宙のどこで起きたバグも手元で再現できます。
2.  **ロジックと表示の完全な分離:** React などの UI フレームワークは、単に `state` を受け取って描画するだけの「皮」になります。
3.  **高速な自動テスト:** ゲームを実際に起動してポチポチ操作しなくても、コアロジックの正しさをミリ秒単位のテストで保証できます。

最初は「すべての副作用を引数で渡すのは冗長だ」と感じるかもしれません。しかし、プロジェクトの規模が大きくなればなるほど、この「純粋さ」がもたらす保守性の高さが、あなたの開発を強力に支えてくれるはずです。

副作用を恐れず、Reducer の中にゲームの宇宙を閉じ込めてみてください。

---

### 次のステップへのヒント

-   **シード可能な乱数生成器:** `seedrandom` ライブラリを使えば、一つの文字列から再現可能な乱数系列を作れます。
-   **Undo/Redo の実装:** 状態がイミュータブルなので、過去の `state` を配列に保存しておくだけで、簡単に「一手戻す」機能が実装できます。
-   **リプレイ機能:** プレイヤーの全 `action` ログを保存すれば、それを再度 Reducer に流し込むだけでゲームのリプレイが再生できます。
