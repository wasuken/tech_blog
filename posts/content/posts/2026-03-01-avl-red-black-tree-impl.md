---
title: "AVL木と赤黒木をPythonで実装して比較する"
date: 2026-03-01T00:00:00+09:00
draft: false
tags: ["algorithm", "data-structure", "python", "red-black-tree", "avl-tree", "技術書"]
categories: ["algorithm"]
description: "AVL木と赤黒木をPythonで実装し、回転・バランス戦略の違いをベンチマークで比較する"
---

## はじめに

[動かしながらゼロから学ぶLinuxカーネルの教科書 第2版](https://info.nikkeibp.co.jp/media/LIN/atcl/books/070900046/)

上記の技術書を読んでいてLinuxスケジューラの話が出てきた。CFSもEEVDFも赤黒木を使っている。赤黒木とは何かを調べていくうちにAVL木との比較が面白かったので、両方Pythonで実装してベンチマークを取った。

実装コードはClaude (Anthropic) により生成。リポジトリは以下。

https://github.com/wasuken/avl-rb-tree

---

## 自己平衡二分探索木とは

まず前提として、ただの二分探索木には問題がある。

```
挿入順: 1, 2, 3, 4, 5

1
 \
  2
   \
    3
     \
      4  ← 一直線になる
```

挿入順によっては木が一直線になり、検索がO(n)に劣化する。これを解決するのが自己平衡二分探索木で、挿入・削除のたびに自動でバランスを取り直す。AVL木と赤黒木はどちらもこのカテゴリに属する。

---

## AVL木

1962年にソ連の数学者Adelson-VelskyとLandis（AVLの名前の由来）が考案した世界初の自己平衡二分探索木。

### バランスの管理方法

各ノードに高さ（height）を持たせ、左右の高さの差（バランス係数）が常に1以内になるよう管理する。

```python
def _balance_factor(self, node):
    return self._height(node.left) - self._height(node.right)
```

差が2以上になったら回転で修正する。

### 回転

回転は「親子関係を1段入れ替えるだけ」の操作。二分探索木の順序を壊さずに形だけ変える。

```
rotate_right(y):

    y              x
   / \            / \
  x   C    →    A   y
 / \               / \
A   t2           t2   C
```

`t2` は回転で行き場を失う孫ノード。二分探索木の順序的に `x < t2 < y` が保証されているので、yの左に付け替えるだけでよい。

バランス崩れのパターンは4つ（LL, RR, LR, RL）で、それぞれ1〜2回の回転で解消できる。

### 挿入・削除

挿入・削除のたびに `_rebalance` が呼ばれ、高さの更新とバランスチェックが走る。

```python
def _rebalance(self, node):
    self._update_height(node)
    bf = self._balance_factor(node)
    if bf > 1:   # Left Heavy
        ...
        return self._rotate_right(node)
    if bf < -1:  # Right Heavy
        ...
        return self._rotate_left(node)
    return node
```

削除は消すノードの子の数によって3パターン。両方の子がある場合は右部分木の最小値（successor）で置き換える。

```python
def _min_node(self, node):
    while node.left:
        node = node.left
    return node
```

`_min_node` がシンプルで面白い。「二分探索木では左に行くほど小さい」というルールを使って、ひたすら左を辿るだけ。

---

## 赤黒木

### AVL木との設計上の違い

AVL木は高さという数値でバランスを管理するが、赤黒木は色（赤/黒）という1bitの情報でバランスを管理する。

```
AVL: height の数値を見てバランス判定
RB:  色のルールを維持することでバランスを保証
```

### 4つのルール

```
1. ノードは赤か黒
2. ルートは必ず黒
3. 赤ノードの子は必ず黒（赤の連続禁止）
4. どのノードからNULLまでの経路の黒ノード数は全経路で同じ
```

4が一番重要で、これが実質的に高さを保証している。黒ノードだけで見れば完全にバランスが取れており、赤は黒の間にしか入れないのでどんなに多くても黒ノードの数を超えられない。結果として高さは `2*log2(n+1)` を超えない。

### 番兵（NIL）

赤黒木はNULLの代わりに番兵ノードを使う。

```python
self.NIL = RBNode(key=None, color=BLACK)
self.root = self.NIL
```

葉ノードの子はすべてこのNILを指す。「黒ノード数が全経路で同じ」というルールをコードで扱いやすくするための実装上の工夫。

### 挿入とfixup

挿入の場所探索はAVL木と同じ。新しいノードを赤で置いた後、色ルール違反が起きていれば `_insert_fixup` で修正する。

fixupのケースは3パターン（叔父ノードの色と挿入位置の組み合わせ）で、色替えだけで済むケースと回転が必要なケースがある。コードが長く見えるのは左右対称で2セットあるから。

---

## ベンチマーク比較

```
              n=1000    n=10000   n=100000
insert AVL    3.00ms    41.70ms   591.12ms
insert RB     1.27ms    16.70ms   289.75ms  ← 約2倍速い

search AVL    0.31ms     0.48ms     1.30ms
search RB     0.49ms     1.11ms     1.87ms  ← ほぼ同じ

delete AVL    1.35ms     1.96ms     2.92ms
delete RB     0.52ms     1.00ms     1.39ms  ← 約2倍速い

height AVL    11 / 16 / 20
height RB     11 / 16 / 20  ← 同じ
```

高さが同じなのに挿入・削除は2倍の差がある。

挿入のたびにAVL木は全ノード辿りながら高さを更新してバランスチェックをするが、赤黒木は色替えだけで済むケースが多く回転が少ない。n=100000で挿入が10万回あれば、1回あたりのわずかな差が積み重なって2倍になる。

まとめると：

```
AVL: 高さという数値を正確に管理するコスト
RB:  色という1bitで高さを間接的に担保するコスト

→ 結果的に高さはほぼ同じ、でも挿入・削除コストに2倍の差
```

Linuxのスケジューラが赤黒木を選ぶ理由がデータで見える結果になった。プロセスのIOのたびに挿入・削除が走るので、挿入・削除の速さが直接レイテンシに影響する。

---

## 参考

- [wasuken/avl-rb-tree - GitHub](https://github.com/wasuken/avl-rb-tree)
- [Linux Kernel Documentation - CFS Scheduler](https://www.kernel.org/doc/html/latest/scheduler/sched-design-CFS.html)
