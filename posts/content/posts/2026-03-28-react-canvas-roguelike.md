---
title: "React + Canvas 2D API で作るターン制ローグライク：論理と描画を切り離す設計"
date: 2026-03-28T13:00:00+09:00
tags: ["React", "Canvas", "ゲーム開発", "TypeScript", "設計"]
categories: ["開発"]
---

# React + Canvas 2D API で作るターン制ローグライク：論理と描画を切り離す設計

React でゲームを作る際、多くの開発者が最初に直面するのが「DOM で描画するか、Canvas で描画するか」という選択です。特に数千枚のタイルや多数のユニットが登場するローグライクゲームでは、DOM 要素の管理はすぐにパフォーマンスの限界に達します。

本稿では、React の強力な状態管理（`useReducer`）と、Canvas 2D API の命令的な描画を組み合わせ、滑らかなアニメーションを実現しつつ堅牢なゲームロジックを維持する設計手法について解説します。

---

## 1. 概要：なぜ React と Canvas を組み合わせるのか

React は「宣言的」な UI 構築に長けていますが、毎秒 60 回の頻度で数千の DOM 要素を更新するような動的な描画には向いていません。一方で、Canvas は「命令的」であり、ピクセル単位での高速な描画が可能ですが、状態と描画の同期を自分で行う必要があります。

この二つの「いいとこ取り」をするのが、**「ロジックは React（useReducer）で、描画は Canvas で」** という役割分担です。

### 本記事で構築するアーキテクチャ
- **Core Logic (`useReducer`)**: ゲームの「真実の状態（State of Truth）」を管理。ターン単位で離散的に変化する座標などを扱う。
- **Render Loop (`requestAnimationFrame`)**: Canvas 上で毎フレーム実行される描画処理。
- **Interpolation (補完)**: 離散的な論理座標を、滑らかな描画座標へと変換するローカル状態管理。

---

## 2. 設計判断：論理座標と描画座標の分離

ローグライクゲームは基本的に「ターン制」です。プレイヤーが右に移動したとき、内部データ（Core State）では `x: 10` から `x: 11` へと一瞬で書き換わります。しかし、これをそのまま描画すると、キャラクターがワープしたように見えてしまいます。

滑らかな移動（アニメーション）を実現するためには、以下の二種類の状態を明確に分ける必要があります。

### なぜ分離が必要か

| 種類 | 管理場所 | 特徴 | 役割 |
| :--- | :--- | :--- | :--- |
| **論理座標 (Logical Position)** | `useReducer` (Global) | 整数（タイル単位）。 | 当たり判定、AI、クエスト進行など。 |
| **描画座標 (Visual Position)** | Canvas 内の Actor クラス (Local) | 小数点を含むピクセル単位。 | 滑らかな移動、揺れ、エフェクト。 |

**判断理由：**
Core State にアニメーションの「途中経過（x: 10.2 など）」を持たせてしまうと、ゲームロジックが描画の都合に汚染されます。例えば、「まだ移動アニメーション中だから攻撃はできない」といった判定をロジック層で書く必要が出てき、コードが複雑化します。ロジック層は常に「今は（論理的に）どこにいるか」だけを知っていれば良いのです。

---

## 3. Canvas Render Loop の実装

React のライフサイクルの中で `requestAnimationFrame` (rAF) を安全に回すために、カスタムフック `useCanvas` を作成します。

### `useCanvas.ts` の実装

```typescript
import { useEffect, useRef } from 'react';

/**
 * Canvas の Render Loop を管理するフック
 * @param draw 毎フレーム実行される描画関数
 */
export const useCanvas = (draw: (ctx: CanvasRenderingContext2D, frameCount: number) => void) => {
  const canvasRef = useRef<HTMLCanvasElement>(null);

  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;

    const context = canvas.getContext('2d');
    if (!context) return;

    let frameCount = 0;
    let animationFrameId: number;

    const render = () => {
      frameCount++;
      draw(context, frameCount);
      animationFrameId = window.requestAnimationFrame(render);
    };

    render();

    // クリーンアップ：コンポーネント消滅時にループを止める
    return () => {
      window.cancelAnimationFrame(animationFrameId);
    };
  }, [draw]);

  return canvasRef;
};
```

**なぜ `useEffect` を使うのか：**
Canvas は命令的な API です。React の宣言的な再レンダリングに描画を任せると、フレームレートが安定せず、また React 自身のオーバーヘッドでカクつきが発生します。`useEffect` 内で独立したループを回すことで、React のレンダリングサイクルとは切り離された 60FPS の安定した描画が可能になります。

---

## 4. `useReducer` との共存：State を Canvas へ渡す

次に、ゲームの状態を管理する `useReducer` を定義し、それを Canvas のループに渡す方法を考えます。

### `game-reducer.ts`（架空のロジック）

```typescript
type Position = { x: number; y: number };

interface GameState {
  player: { pos: Position; hp: number };
  enemies: Array<{ id: string; pos: Position; type: string }>;
  map: number[][]; // タイルID
}

type Action = 
  | { type: 'MOVE_PLAYER'; delta: Position }
  | { type: 'TICK_TURN' };

export const gameReducer = (state: GameState, action: Action): GameState => {
  switch (action.type) {
    case 'MOVE_PLAYER':
      const newPos = { 
        x: state.player.pos.x + action.delta.x, 
        y: state.player.pos.y + action.delta.y 
      };
      // ここで一気に座標が書き換わる（ワープ状態）
      return { ...state, player: { ...state.player, pos: newPos } };
    default:
      return state;
  }
};
```

### `useRef` によるブリッジ

ここで問題になるのが、`useCanvas` に渡す `draw` 関数の中で最新の `state` をどう参照するかです。`draw` 関数を `state` が変わるたびに更新すると、`useEffect` が再実行され、ループがリセットされてしまいます。

これを避けるために、**`useRef` を使って最新の State を保持する**テクニックを使います。

```tsx
const FieldScreen = () => {
  const [state, dispatch] = useReducer(gameReducer, initialState);
  
  // 最新のstateを常にrefに同期させる
  const stateRef = useRef(state);
  useEffect(() => {
    stateRef.current = state;
  }, [state]);

  // draw 関数は useCallback で固定し、内部で stateRef を見る
  const draw = useCallback((ctx: CanvasRenderingContext2D, frameCount: number) => {
    const currentState = stateRef.current;
    
    // 描画処理
    ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height);
    renderGame(ctx, currentState);
  }, []);

  const canvasRef = useCanvas(draw);

  return <canvas ref={canvasRef} width={800} height={600} />;
};
```

**なぜ `useRef` なのか：**
`useRef` の `.current` は書き換えても再レンダリングを発生させません。これにより、「React が管理する最新の状態」を「Canvas の独立した描画ループ」から安全に、かつ最新の状態で読み出すことができます。

---

## 5. 敵 Actor のローカル State 管理：補完アニメーション

前述の通り、`state.enemies[0].pos` はターンごとに `(10, 10)` から `(11, 10)` へ飛びます。これを滑らかに見せるために、描画層専用の `Actor` クラスを導入します。

### Actor クラスの実装

```typescript
class Actor {
  id: string;
  visualX: number; // 描画用の浮動小数点座標
  visualY: number;
  lerpSpeed = 0.15; // 追従速度

  constructor(id: string, initialPos: Position) {
    this.id = id;
    this.visualX = initialPos.x;
    this.visualY = initialPos.y;
  }

  /**
   * 毎フレーム実行される更新処理
   * @param targetPos ロジック層（Core State）の論理座標
   */
  update(targetPos: Position) {
    // 線形補完 (Lerp) を用いて、現在の描画座標を目標座標に近づける
    this.visualX += (targetPos.x - this.visualX) * this.lerpSpeed;
    this.visualY += (targetPos.y - this.visualY) * this.lerpSpeed;
  }

  draw(ctx: CanvasRenderingContext2D, offsetX: number, offsetY: number, tileSize: number) {
    const screenX = this.visualX * tileSize + offsetX;
    const screenY = this.visualY * tileSize + offsetY;
    
    // キャラクターの描画
    ctx.fillStyle = 'red';
    ctx.fillRect(screenX, screenY, tileSize, tileSize);
  }
}
```

### 描画ループ内での Actor 管理

```typescript
// コンポーネント外、または useRef で Actor の Map を保持
const actorsRef = useRef<Map<string, Actor>>(new Map());

const renderGame = (ctx: CanvasRenderingContext2D, state: GameState) => {
  const TILE_SIZE = 32;
  const actors = actorsRef.current;

  // 1. 存在しない Actor を追加、不要なものを削除
  syncActors(actors, state.enemies);

  // 2. 更新と描画
  state.enemies.forEach(enemy => {
    const actor = actors.get(enemy.id);
    if (actor) {
      actor.update(enemy.pos); // ここで論理座標に向かって滑らかに動く
      actor.draw(ctx, 0, 0, TILE_SIZE);
    }
  });
};
```

**なぜ `lerp` (線形補完) なのか：**
`lerp` はシンプルながら非常に強力です。ターゲットとの距離に比例して移動速度が変わるため、動き始めが速く、停止直前がゆっくりになる「イージング」の効果が自然に得られます。また、通信遅延や処理落ちで論理座標の更新が飛んでも、描画座標はそれを追いかける形になるため、見た目上のガタつきを最小限に抑えられます。

---

## 6. タイルベース描画の最適化：カメラとクリッピング

最後に、広大なマップを効率よく描画する手法についてです。

### カメラオフセットの計算

プレイヤーが常に画面中央に来るように描画位置をずらします。

```typescript
const getCameraOffset = (ctx: CanvasRenderingContext2D, playerVisualX: number, playerVisualY: number, tileSize: number) => {
  return {
    x: ctx.canvas.width / 2 - (playerVisualX * tileSize + tileSize / 2),
    y: ctx.canvas.height / 2 - (playerVisualY * tileSize + tileSize / 2)
  };
};
```

### ビューポートクリッピング（間引き描画）

画面外にあるタイルを描画するのは CPU/GPU の無駄遣いです。描画範囲を計算してループを回します。

```typescript
const drawMap = (ctx: CanvasRenderingContext2D, state: GameState, offset: {x: number, y: number}, tileSize: number) => {
  const viewWidth = ctx.canvas.width;
  const viewHeight = ctx.canvas.height;

  // 画面内に収まるタイルのインデックス範囲を計算
  const startCol = Math.floor(-offset.x / tileSize);
  const endCol = startCol + Math.ceil(viewWidth / tileSize);
  const startRow = Math.floor(-offset.y / tileSize);
  const endRow = startRow + Math.ceil(viewHeight / tileSize);

  for (let r = startRow; r <= endRow; r++) {
    for (let c = startCol; c <= endCol; c++) {
      const tileId = state.map[r]?.[c];
      if (tileId === undefined) continue;

      const x = c * tileSize + offset.x;
      const y = r * tileSize + offset.y;
      
      // ここでタイル画像を描画
      // ctx.drawImage(tileAtlas, ...);
    }
  }
};
```

**判断理由：**
Canvas の `drawImage` は高速ですが、数万回の呼び出しは流石に重くなります。クリッピングを実装することで、たとえ 1000x1000 の広大なマップであっても、描画負荷は常に「画面解像度分」に固定されます。これはローグライクにおけるパフォーマンス最適化の基本です。

---

## 7. まとめ

React と Canvas を用いたゲーム開発では、**「論理の React」と「描画の Canvas」をどう繋ぐか**が最大のポイントです。

- **`useReducer`** で純粋なゲームの状態を管理する。
- **`useRef`** でその状態を Canvas のループへ橋渡しする。
- **Actor クラス** に描画専用のローカル状態（補完座標）を持たせ、滑らかなアニメーションを実現する。
- **カメラとクリッピング** で描画負荷を最小限に抑える。

この設計にすることで、React の高い生産性と、Canvas の圧倒的な描画パフォーマンスを両立させることができます。また、アニメーションのロジックが描画層に閉じ込められているため、将来的に「キャラをジャンプさせたい」「ダメージ時に画面を揺らしたい」といった要望が出ても、ゲームのコアロジックを一切書き換えることなく対応が可能です。

ぜひ、あなたのローグライク開発にもこの「分離の美学」を取り入れてみてください。
