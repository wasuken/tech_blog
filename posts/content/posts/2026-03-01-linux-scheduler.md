---
title: "LinuxのCFSとEEVDFを整理する - スケジューラはなぜ赤黒木を使うのか"
date: 2026-03-01T00:00:00+09:00
draft: false
tags: ["linux", "kernel", "scheduler", "infrastructure", "sre"]
categories: ["infrastructure"]
description: "CFSのvruntime・赤黒木・タイムスライスの仕組みと、Linux 6.6で導入されたEEVDFへの変遷をSRE視点で整理する"
---

## はじめに

[動かしながらゼロから学ぶLinuxカーネルの教科書 第2版](https://info.nikkeibp.co.jp/media/LIN/atcl/books/070900046/)

上記の技術書を読んでいてスケジューラ周りの理解が曖昧だったので、生成AIや公式ドキュメントを使って整理した。

---

## CFS (Completely Fair Scheduler) とは

Linux 2.6.23から導入されたプロセススケジューラ。「全プロセスに公平にCPU時間を与える」という思想で設計されている。

---

## vruntime（仮想実行時間）

vruntimeは「実際の実行時間をNICE値で補正した値」で、CFSの核心となる指標。
```
vruntime += 実際のCPU時間 × (1024 / プロセスの重み)
```

- NICE値が低い（優先度高）→ 重みが大きい → vruntimeの増加が遅い → より長くCPUを使える
- NICE値が高い（優先度低）→ 重みが小さい → vruntimeの増加が速い → すぐ交代させられる

CFSは「vruntimeが最も小さいプロセスを次に実行する」というルールで動く。**後ろ向きの指標**（過去の使用量の累積）であることがEEVDFとの本質的な差になる。

---

## NICE値と重み

NICE値は `-20`（最高優先度）〜 `+19`（最低優先度）の範囲で、内部的に重みに変換される。
```
NICE  0  → weight 1024
NICE -1  → weight 1277（約1.25倍）
NICE +1  → weight 820（約0.8倍）
NICE -20 → weight 88761
NICE +19 → weight 15
```

1段階変わるごとに約10%のCPU時間が変化する設計になっている。

---

## タイムスライスとスケジューリングレイテンシ

**スケジューリングレイテンシ**は「全プロセスが最低1回実行されるべき目標周期」。デフォルト約6〜24ms（プロセス数による）。

タイムスライスはその比例配分：
```
タイムスライス = スケジューリングレイテンシ × (タスクの重み / キュー内の全タスクの重みの合計)
```

具体例：
```
レイテンシ = 12ms
プロセスA: NICE -5 → weight 3121
プロセスB: NICE +5 → weight 335
合計 = 3456

A: 12ms × (3121 / 3456) ≈ 10.8ms
B: 12ms × (335  / 3456) ≈  1.2ms
```

「優先度が高いほど多くもらえるが、全員必ず実行される」がCFSの設計思想。優先度で独占させないのはスタベーション（低優先度プロセスが永久に実行されない）を防ぐため。

---

## 赤黒木（Red-Black Tree）

CFSとEEVDF両方で使われるデータ構造。「自己平衡二分探索木」のひとつ。

### なぜただの二分探索木ではダメか

挿入順によっては木が一直線になりO(n)に劣化する。
```
挿入: 1, 2, 3, 4, 5

1
 \
  2
   \
    3  ← 検索がO(n)になる
```

### 赤黒木のバランス保証

ノードに赤/黒の色をつけ、以下のルールを維持することで自動的にバランスを保つ：

1. ノードは赤か黒
2. ルートは黒
3. 赤ノードの子は必ず黒（赤の連続禁止）
4. どのノードからNULLまでの黒ノード数は全経路で同じ

挿入・削除のたびに回転と色変更が自動で走り、常にO(log n)を保証する。

| | 最悪ケース |
|--|--|
| ただの二分探索木 | O(n) |
| 赤黒木 | O(log n) 保証 |

### スケジューラが赤黒木を選ぶ理由

プロセスはIOを待ち始めると木から取り出され、IO完了で木に戻る。この挿入・削除がミリ秒単位で頻繁に発生するため、**検索の精度より挿入・削除の速度**が重要になる。

AVL木は検索が速い代わりに挿入・削除のコストが高く、スケジューラのユースケースには合わない。赤黒木はその逆のトレードオフを持つ。

---

## EEVDF (Earliest Eligible Virtual Deadline First)

Linux 6.6（2023年）で導入。元は1995年の論文のアルゴリズム。

### CFSの限界

vruntimeは後ろ向きの指標なので「次にいつ実行されるか」という前向きの予測ができない。レイテンシの保証が困難だった。

### EEVDFの2つの核心概念

**eligible time（実行資格時刻）**：タスクがCPUをもらう権利を得る時刻。これより前は実行されない。

**virtual deadline**：タスクが「このvruntime時点までには実行されるべき」という期限。
```
virtual deadline = eligible time + タイムスライス
```

### 選択ルールの違い
```
CFS:   vruntimeが最小のタスクを選ぶ
EEVDF: eligibleなタスクの中でvirtual deadlineが最も早いタスクを選ぶ
```

### タイムスライスの要求

EEVDFではタスクが希望するタイムスライス長を伝えられる：
```
レイテンシ重視タスク → 短いタイムスライスを要求 → 頻繁に実行される
スループット重視タスク → 長いタイムスライスを要求 → まとめて実行される
```

CFSはこの区別ができなかった。

### 実装はCFSとほぼ同じ

赤黒木はそのまま使い、ソートキーだけ変更：
```
CFS:   key = vruntime
EEVDF: key = virtual_deadline
```

`kernel/sched/fair.c` の中にCFSとEEVDFの実装が共存しており、Linux 6.6でCFSのコードベースを拡張する形で導入された。

---

## CFSとEEVDFの対比
```
CFS:
  過去の使用量(vruntime)で順番を決める
  → 公平だが「いつ実行されるか」の保証なし

EEVDF:
  未来の期限(virtual deadline)で順番を決める
  → 公平さを保ちつつ待ち時間の上限を保証できる
```

---

## 参考

- [Linux Kernel Documentation - Completely Fair Scheduler](https://www.kernel.org/doc/html/latest/scheduler/sched-design-CFS.html)
- [Linux Kernel Documentation - EEVDF Scheduler](https://www.kernel.org/doc/html/latest/scheduler/sched-eevdf.html)
- [LWN.net - An EEVDF CPU scheduler for Linux](https://lwn.net/Articles/925371/)
- [論文: Earliest Eligible Virtual Deadline First (1995)](https://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.46.2706)
