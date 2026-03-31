---
title: "Canvas 2D API でピクセルアートタイルを「手書き」する実装テクニック"
date: 2026-03-31T09:00:00+09:00
tags: ["Canvas", "JavaScript", "ゲーム開発", "ピクセルアート"]
categories: ["開発"]
---

# Canvas 2D API でピクセルアートタイルを「手書き」する実装テクニック

## 1. 概要

Web ゲーム開発、特にローグライクやタクティカル RPG を開発する際、避けて通れないのが「マップの描画」だ。通常、これらは「タイルセット」と呼ばれる画像ファイルを読み込んで描画するが、開発の初期段階や、あえて外部アセットに頼りたくない場合、Canvas 2D API を使ってプログラムで直接タイルを描画する「手書き（プログラマティック描画）」の手法が非常に強力な武器になる。

本記事では、HTML5 Canvas の `fillRect` や `beginPath` などの基本命令のみを使い、擬似的なピクセルアート風のタイルを描画するテクニックを解説する。画像を用意する手間を省きつつ、動的に色や形状を変更できる柔軟な描画システムを構築しよう。

---

## 2. タイル描画の基本構造

まずは、どのようなタイルでも共通して利用できる描画の入り口を作る。ピクセル座標ではなく、タイル座標（x, y）とタイルサイズ（tileSize）を受け取る設計にすることで、グリッドベースのシステムと統合しやすくなる。

```javascript
/**
 * タイルを描画するメイン関数
 * @param {CanvasRenderingContext2D} ctx - Canvasコンテキスト
 * @param {string} tileType - タイルの種類 ('wall', 'floor', 'grass', etc.)
 * @param {number} x - タイルのX座標（グリッド単位）
 * @param {number} y - タイルのY座標（グリッド単位）
 * @param {number} tileSize - 1タイルのピクセルサイズ
 * @param {string} fieldType - フィールドの種類 ('meadow', 'forest', 'mountain')
 */
function drawTile(ctx, tileType, x, y, tileSize, fieldType = 'meadow') {
    const px = x * tileSize;
    const py = y * tileSize;

    ctx.save();
    ctx.translate(px, py);

    switch (tileType) {
        case 'floor':
            drawFloor(ctx, tileSize, fieldType);
            break;
        case 'wall':
            drawWall(ctx, tileSize, fieldType);
            break;
        case 'object_grass':
            drawFloor(ctx, tileSize, fieldType);
            drawGrass(ctx, tileSize);
            break;
        case 'object_tree':
            drawFloor(ctx, tileSize, fieldType);
            drawTree(ctx, tileSize);
            break;
        case 'stairs_down':
            drawFloor(ctx, tileSize, fieldType);
            drawStairs(ctx, tileSize, false);
            break;
        default:
            ctx.fillStyle = '#333';
            ctx.fillRect(0, 0, tileSize, tileSize);
    }

    ctx.restore();
}
```

この設計のポイントは、`ctx.translate` を使ってタイルの左上を原点 (0, 0) に固定することだ。これにより、各タイルの描画ロジック内で座標計算を簡略化できる。

---

## 3. タイルタイプ別描画の実装

### 3.1 床（Floor）とフィールドタイプによる変化

床は最も描画回数が多いタイルだ。単色で塗るのではなく、フィールドのタイプによってベースカラーを変え、わずかなノイズ（ドット）を加えることで「ピクセルアート感」を出す。

```javascript
function drawFloor(ctx, tileSize, fieldType) {
    let baseColor, noiseColor;

    switch (fieldType) {
        case 'forest':
            baseColor = '#2d4c1e';
            noiseColor = '#243b18';
            break;
        case 'mountain':
            baseColor = '#5a5a5a';
            noiseColor = '#4a4a4a';
            break;
        case 'meadow':
        default:
            baseColor = '#4a773c';
            noiseColor = '#3d6231';
    }

    ctx.fillStyle = baseColor;
    ctx.fillRect(0, 0, tileSize, tileSize);

    ctx.fillStyle = noiseColor;
    const dotSize = tileSize / 8;
    ctx.fillRect(dotSize * 2, dotSize * 1, dotSize, dotSize);
    ctx.fillRect(dotSize * 5, dotSize * 4, dotSize, dotSize);
    ctx.fillRect(dotSize * 1, dotSize * 6, dotSize, dotSize);
}
```

### 3.2 壁（Wall）

壁は奥行きを感じさせることが重要だ。上面と前面で色を変え、ハイライトを入れることで立体感を演出する。

```javascript
function drawWall(ctx, tileSize, fieldType) {
    const topColor = fieldType === 'mountain' ? '#888' : '#5d4037';
    const sideColor = fieldType === 'mountain' ? '#555' : '#3e2723';
    const highlight = '#ffffff33';

    ctx.fillStyle = sideColor;
    ctx.fillRect(0, 0, tileSize, tileSize);

    ctx.fillStyle = topColor;
    ctx.fillRect(0, 0, tileSize, tileSize * 0.8);

    ctx.fillStyle = highlight;
    ctx.fillRect(0, 0, tileSize, 2);
    ctx.fillRect(0, 0, 2, tileSize * 0.8);

    ctx.fillStyle = '#00000022';
    ctx.fillRect(tileSize * 0.5, tileSize * 0.2, 2, tileSize * 0.4);
    ctx.fillRect(0, tileSize * 0.5, tileSize, 2);
}
```

### 3.3 オブジェクト：木（Tree）と草（Grass）

オブジェクトは `beginPath` を活用して形状を作る。

```javascript
function drawTree(ctx, tileSize) {
    const unit = tileSize / 10;

    ctx.fillStyle = '#4e342e';
    ctx.fillRect(unit * 4, unit * 6, unit * 2, unit * 4);

    ctx.fillStyle = '#2e7d32';
    ctx.beginPath();
    ctx.moveTo(unit * 1, unit * 7);
    ctx.lineTo(unit * 9, unit * 7);
    ctx.lineTo(unit * 5, unit * 3);
    ctx.fill();

    ctx.fillStyle = '#388e3c';
    ctx.beginPath();
    ctx.moveTo(unit * 2, unit * 4);
    ctx.lineTo(unit * 8, unit * 4);
    ctx.lineTo(unit * 5, unit * 1);
    ctx.fill();
}

function drawGrass(ctx, tileSize) {
    const unit = tileSize / 8;
    ctx.strokeStyle = '#8bc34a';
    ctx.lineWidth = 2;

    ctx.beginPath();
    ctx.moveTo(unit * 2, unit * 7);
    ctx.lineTo(unit * 1, unit * 4);
    ctx.moveTo(unit * 4, unit * 7);
    ctx.lineTo(unit * 4, unit * 3);
    ctx.moveTo(unit * 6, unit * 7);
    ctx.lineTo(unit * 7, unit * 4);
    ctx.stroke();
}
```

### 3.4 階段（Stairs）

階段はローグライクにおける重要な要素だ。コントラストを強めにして、プレイヤーがすぐに見つけられるようにする。

```javascript
function drawStairs(ctx, tileSize, isUp = false) {
    const stepCount = 4;
    const stepHeight = tileSize / stepCount;

    for (let i = 0; i < stepCount; i++) {
        const shade = 150 - (i * 20);
        ctx.fillStyle = `rgb(${shade}, ${shade}, ${shade})`;

        if (isUp) {
            ctx.fillRect(0, tileSize - (i + 1) * stepHeight, tileSize, stepHeight);
        } else {
            ctx.fillRect(i * (tileSize / stepCount / 2), i * stepHeight, tileSize - i * (tileSize / stepCount), stepHeight);
        }
    }
}
```

---

## 4. フォールバック設計としての活用

なぜ画像を使わずにこのような手間をかけるのか。その最大の理由は「開発効率」と「柔軟性」だ。

1. **プロトタイピングの高速化**: デザイナーがアセットを完成させるのを待つ必要がない。
2. **動的なバリエーション**: `fieldType` を変えるだけで、草原、砂漠、雪原などのバリエーションを無限に作れる。
3. **アセット未発見時の回避策**: ロードエラーが発生した際や、特定のタイル画像がまだ存在しない場合のフォールバックとして使える。

```javascript
function renderMap(mapData, assets) {
    mapData.forEach((tile) => {
        if (assets[tile.type]) {
            ctx.drawImage(assets[tile.type], tile.x * TILE_SIZE, tile.y * TILE_SIZE);
        } else {
            drawTile(ctx, tile.type, tile.x, tile.y, TILE_SIZE, mapData.environment);
        }
    });
}
```

---

## 5. 画像（PNG）との使い分け

| 特徴 | Canvas 手書き | 画像（PNG/Sprite） |
| :--- | :--- | :--- |
| **制作コスト** | 低（コードのみ） | 高（ペイントソフトが必要） |
| **表現力** | 限定的（幾何学的） | 無限（ディテールが凝れる） |
| **カスタマイズ** | 容易（変数を変えるだけ） | 困難（色違い画像を量産） |
| **パフォーマンス** | 計算量に依存 | 転送量に依存 |

結論として、**「ベースの地面や壁は Canvas で動的に描き、キャラや重要なボス、凝ったエフェクトには画像を使う」** というハイブリッドな構成が、インディーゲーム開発においては非常にバランスが良い選択だ。

---

## 6. `save/restore` と `translate` を理解する

記事を読んで気になった点をまとめておく。

### `translate` はなぜ便利なのか

最初、「なんで元々原点を(0,0)に固定しとらんねん」と思った。

答えは**グリッド座標からピクセル座標への変換を1回で済ませるため**だ。`translate` しないと各 `fillRect` の中で毎回 `x * tileSize + ...` って計算しないといけない。`translate` で原点をずらしておけば、あとは `(0, 0)` 基準で描くだけでいい。

### `save/restore` は中間地点とロールバック

`save` と `restore` はセットで使う。

- `save` → 今の状態をスタックに積む（中間地点を記録）
- `restore` → 積んだ状態に戻す（ロールバック）

```javascript
ctx.translate(0, 0);     // 原点はここ
ctx.save();              // この状態を保存
ctx.translate(100, 100); // 原点を移動して描画
// ...
ctx.restore();           // saveした時点に戻る → 原点は(0,0)に戻る
```

`restore` だけじゃ「どこに戻るか」がわからないので `save` が必要だ。タイル1個描くたびに座標をリセットできる仕組みで、CSS の `transform` と同じ発想。

---

## 7. まとめ

Canvas 2D API を使ったタイルの手書き描画は、一見すると地味な作業だが、マスターすればゲーム開発の自由度が飛躍的に向上する。

- `fillRect` で基本的な色面を作る
- `translate` と `save/restore` で座標系を整理する
- フィールドタイプごとに色定数を切り替える
- わずかなハイライトとシャドウで立体感を出す

これらのテクニックを組み合わせることで、画像アセットが一切なくても、十分に「ゲームらしい」画面を作り上げることが可能だ。
