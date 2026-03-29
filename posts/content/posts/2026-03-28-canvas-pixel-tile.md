---
title: "Canvas 2D API でピクセルアートタイルを「手書き」する実装テクニック"
date: 2026-03-28T09:00:00+09:00
tags: ["Canvas", "JavaScript", "ゲーム開発", "ピクセルアート"]
categories: ["開発"]
---

# Canvas 2D API でピクセルアートタイルを「手書き」する実装テクニック

## 1. 概要

Web ゲーム開発、特にローグライクやタクティカル RPG を開発する際、避けて通れないのが「マップの描画」です。通常、これらは「タイルセット」と呼ばれる画像ファイルを読み込んで描画しますが、開発の初期段階や、あえて外部アセットに頼りたくない場合、Canvas 2D API を使ってプログラムで直接タイルを描画する「手書き（プログラマティック描画）」の手法が非常に強力な武器になります。

本記事では、HTML5 Canvas の `fillRect` や `beginPath` などの基本命令のみを使い、擬似的なピクセルアート風のタイルを描画するテクニックを解説します。画像を用意する手間を省きつつ、動的に色や形状を変更できる柔軟な描画システムを構築しましょう。

---

## 2. タイル描画の基本構造

まずは、どのようなタイルでも共通して利用できる描画の入り口を作ります。ピクセル座標ではなく、タイル座標（x, y）とタイルサイズ（tileSize）を受け取る設計にすることで、グリッドベースのシステムと統合しやすくなります。

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

    // 描画状態の保存
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
            drawFloor(ctx, tileSize, fieldType); // 下地に床を描く
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
            // 未定義のタイルは塗りつぶし
            ctx.fillStyle = '#333';
            ctx.fillRect(0, 0, tileSize, tileSize);
    }

    ctx.restore();
}
```

この設計のポイントは、`ctx.translate` を使ってタイルの左上を原点 (0, 0) に固定することです。これにより、各タイルの描画ロジック内で座標計算を簡略化できます。

---

## 3. タイルタイプ別描画の実装

### 3.1 床（Floor）とフィールドタイプによる変化

床は最も描画回数が多いタイルです。単色で塗るのではなく、フィールドのタイプによってベースカラーを変え、わずかなノイズ（ドット）を加えることで「ピクセルアート感」を出します。

```javascript
function drawFloor(ctx, tileSize, fieldType) {
    let baseColor, noiseColor;

    switch (fieldType) {
        case 'forest':
            baseColor = '#2d4c1e'; // 深緑
            noiseColor = '#243b18';
            break;
        case 'mountain':
            baseColor = '#5a5a5a'; // 岩の色
            noiseColor = '#4a4a4a';
            break;
        case 'meadow':
        default:
            baseColor = '#4a773c'; // 草原
            noiseColor = '#3d6231';
    }

    // ベース
    ctx.fillStyle = baseColor;
    ctx.fillRect(0, 0, tileSize, tileSize);

    // 簡易的なテクスチャ（ドット）
    ctx.fillStyle = noiseColor;
    const dotSize = tileSize / 8;
    ctx.fillRect(dotSize * 2, dotSize * 1, dotSize, dotSize);
    ctx.fillRect(dotSize * 5, dotSize * 4, dotSize, dotSize);
    ctx.fillRect(dotSize * 1, dotSize * 6, dotSize, dotSize);
}
```

### 3.2 壁（Wall）

壁は奥行きを感じさせることが重要です。上面と前面で色を変え、ハイライトを入れることで立体感を演出します。

```javascript
function drawWall(ctx, tileSize, fieldType) {
    const topColor = fieldType === 'mountain' ? '#888' : '#5d4037';
    const sideColor = fieldType === 'mountain' ? '#555' : '#3e2723';
    const highlight = '#ffffff33';

    // 全体を側面の色で塗る
    ctx.fillStyle = sideColor;
    ctx.fillRect(0, 0, tileSize, tileSize);

    // 上面を描画
    ctx.fillStyle = topColor;
    ctx.fillRect(0, 0, tileSize, tileSize * 0.8);

    // ハイライト（エッジ）
    ctx.fillStyle = highlight;
    ctx.fillRect(0, 0, tileSize, 2); // 上辺
    ctx.fillRect(0, 0, 2, tileSize * 0.8); // 左辺

    // レンガ状の模様
    ctx.fillStyle = '#00000022';
    ctx.fillRect(tileSize * 0.5, tileSize * 0.2, 2, tileSize * 0.4);
    ctx.fillRect(0, tileSize * 0.5, tileSize, 2);
}
```

### 3.3 オブジェクト：木（Tree）と草（Grass）

オブジェクトは `beginPath` を活用して形状を作ります。アンチエイリアスがかからないように注意するか、あえて小さな `fillRect` の集合体として描くことで、よりピクセルアートに近い質感を出すことができます。

```javascript
function drawTree(ctx, tileSize) {
    const unit = tileSize / 10;

    // 幹
    ctx.fillStyle = '#4e342e';
    ctx.fillRect(unit * 4, unit * 6, unit * 2, unit * 4);

    // 葉（三角形を重ねる）
    ctx.fillStyle = '#2e7d32';
    
    // 下段
    ctx.beginPath();
    ctx.moveTo(unit * 1, unit * 7);
    ctx.lineTo(unit * 9, unit * 7);
    ctx.lineTo(unit * 5, unit * 3);
    ctx.fill();

    // 上段
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
    // 左の葉
    ctx.moveTo(unit * 2, unit * 7);
    ctx.lineTo(unit * 1, unit * 4);
    // 中央の葉
    ctx.moveTo(unit * 4, unit * 7);
    ctx.lineTo(unit * 4, unit * 3);
    // 右の葉
    ctx.moveTo(unit * 6, unit * 7);
    ctx.lineTo(unit * 7, unit * 4);
    ctx.stroke();
}
```

### 3.4 階段（Stairs）

階段はローグライクにおける重要な要素です。コントラストを強めにして、プレイヤーがすぐに見つけられるようにします。

```javascript
function drawStairs(ctx, tileSize, isUp = false) {
    const stepCount = 4;
    const stepHeight = tileSize / stepCount;
    
    ctx.fillStyle = '#9e9e9e';
    for (let i = 0; i < stepCount; i++) {
        const shade = 150 - (i * 20);
        ctx.fillStyle = `rgb(${shade}, ${shade}, ${shade})`;
        
        if (isUp) {
            // 上り階段：下から上へ
            ctx.fillRect(0, tileSize - (i + 1) * stepHeight, tileSize, stepHeight);
        } else {
            // 下り階段：上から下へ（奥に向かって沈むイメージ）
            ctx.fillRect(i * (tileSize / stepCount / 2), i * stepHeight, tileSize - i * (tileSize / stepCount), stepHeight);
        }
    }
}
```

---

## 4. フォールバック設計としての活用

なぜ画像を使わずにこのような手間をかけるのでしょうか？ その最大の理由は「開発効率」と「柔軟性」です。

1.  **プロトタイピングの高速化**: デザイナーがアセットを完成させるのを待つ必要がありません。
2.  **動的なバリエーション**: `fieldType` を変えるだけで、草原、砂漠、雪原などのバリエーションを無限に作れます。
3.  **アセット未発見時の回避策**: ロードエラーが発生した際や、特定のタイル画像がまだ存在しない場合のフォールバックとして `drawTile` を呼び出すようにしておけば、ゲームがクラッシュしたり、真っ黒な画面になったりするのを防げます。

```javascript
// 実践的な描画ループの例
function renderMap(mapData, assets) {
    mapData.forEach((tile) => {
        if (assets[tile.type]) {
            // 画像があれば画像を描画
            ctx.drawImage(assets[tile.type], tile.x * TILE_SIZE, tile.y * TILE_SIZE);
        } else {
            // 画像がなければ「手書き」で描画
            drawTile(ctx, tile.type, tile.x, tile.y, TILE_SIZE, mapData.environment);
        }
    });
}
```

---

## 5. 画像（PNG）との使い分け

もちろん、すべての描画を Canvas 命令で行うのが最適とは限りません。

| 特徴 | Canvas 手書き | 画像（PNG/Sprite） |
| :--- | :--- | :--- |
| **制作コスト** | 低（コードのみ） | 高（ペイントソフトが必要） |
| **表現力** | 限定的（幾何学的） | 無限（ディテールが凝れる） |
| **カスタマイズ** | 容易（変数を変えるだけ） | 困難（色違い画像を量産） |
| **パフォーマンス** | 計算量に依存（複雑だと重い） | 転送量に依存（枚数が多いと重い） |

結論として、**「ベースの地面や壁は Canvas で動的に描き、キャラや重要なボス、凝ったエフェクトには画像を使う」**というハイブリッドな構成が、インディーゲーム開発においては非常にバランスが良い選択と言えます。

---

## 6. まとめ

Canvas 2D API を使ったタイルの手書き描画は、一見すると地味な作業ですが、マスターすればゲーム開発の自由度が飛躍的に向上します。

- `fillRect` で基本的な色面を作る
- `translate` と `save/restore` で座標系を整理する
- フィールドタイプごとに色定数を切り替える
- わずかなハイライトとシャドウで立体感を出す

これらのテクニックを組み合わせることで、画像アセットが一切なくても、十分に「ゲームらしい」画面を作り上げることが可能です。ぜひ、自分のプロジェクトでも `drawTile` 関数を実装して、独自のピクセルワールドを構築してみてください。
