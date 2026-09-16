---
title: "AIに丸投げしたコードを見直したら、SELECTだけで死ぬほど遅かった話"
date: 2026-09-16T12:39:09+09:00
draft: false
tags: ["パフォーマンス", "React", "TypeScript", "PostgreSQL"]
categories: ["技術"]
ai_assisted: true
description: "AIで生成したSQLを調査して直した。今後の展望も少し書く"
---

## 前提

個人開発のEDINET分析ツール（財務レポートを解析してDiscordに投げる趣味プロジェクト）で、フロントエンドのコードをAIにほぼ丸投げして書かせた部分があった。

後から見直していたら、明らかに厳しい実装が出てきたので調査した記録。今回は原因の特定と応急処置（SELECT修正）までで、DBスキーマ自体の見直しは今後の課題として持ち越す。

## 出てきたコード

API利用状況を表示するダッシュボードコンポーネントで、以下のように4本のAPIを`Promise.all`で並列取得していた。

```js
const [statsRes, logsRes, tsRes, statusRes] = await Promise.all([
  apiClient.get(`/api-logs/stats?days=${days}`),
  apiClient.get('/api-logs/logs?limit=50'),
  apiClient.get(`/api-logs/timeseries?days=${days}&interval=${interval}`),
  apiClient.get(`/api-logs/status?days=${days}`),
])
```

取得後、4つのstateを`finally`ブロックで一括更新し、`loading`も1つのフラグで管理していた。

## 最初の疑問：Promise.allの並列度は妥当か

`Promise.all`自体は4本のリクエストを同時発火するので、直列で`await`を重ねる実装([MDN: Promise.all](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Promise/all))と比べれば理にかなっている。

ただし設計上の課題は残る。

- `loading`が1つのフラグなので、一番遅いレスポンスに画面全体の表示タイミングが引っ張られる
- `useEffect`の依存配列が`[days, interval]`のみで、`interval`しか変えていない操作でも無関係な`logs`まで再取得される

対策案としては、各エンドポイントを個別の`useEffect`＋個別`state`に分割し、届いた順にレンダリングする方式が考えられる。この場合、依存配列も自然に分離でき、`interval`変更時に不要な再取得も防げる。ただしこれは「体感速度」の改善であり、根本的なネットワーク・サーバー負荷の解消にはならない。

## 実測結果

Chrome DevToolsのNetworkタブで実測したところ、以下の結果になった。

| エンドポイント | サイズ | レスポンスタイム(1回目) | レスポンスタイム(2回目) |
|---|---|---|---|
| `stats?days=7` | 0.3kB | 5.69秒 | 6.25秒 |
| `logs?limit=50` | 7.7kB | 200ms | 351ms |
| `timeseries?days=7&interval=hour` | 14.5kB | 11.52秒 | 6.01秒 |
| `status?days=7` | 0.5kB | 7.92秒 | 11.58秒 |

同一アプリの別画面（Stats.tsx）にある`summary`・`alerts`・`ranking`系のエンドポイントは、同程度の小さいペイロードで30〜111msと桁違いに速い。

## 結論：問題はフロントの並列化ではなくバックエンド側

数百バイト〜十数キロバイトのレスポンスに対して秒単位〜十数秒のレイテンシは、ネットワークやReact側の描画方式では説明がつかない差だった。並列fetch自体は問題ではなく、疑うべきは以下。

1. **集計クエリのN+1**（プロバイダーごとにループでクエリを投げている可能性）
2. **日時カラムへのインデックス未設定**（`days`によるWHERE句がフルスキャンになっている可能性）
3. **`interval=hour`の集計処理**（DB側の集計かアプリ側ループかで原因が分岐）
4. **2回目の計測でむしろ悪化**（`stats`5.69→6.25秒、`status`7.92→11.58秒）していることから、キャッシュが効いていない、あるいはテーブルの成長に伴って劣化する設計になっている可能性

N+1問題自体については[N+1 query problem (MDN Glossary)](https://developer.mozilla.org/en-US/docs/Glossary/N%2B1_query_problem)を参照。

## 今回の対応範囲

今回は該当エンドポイントのSQLクエリを見直し、集計方法をSELECT文レベルで修正するところまでを実施した。

DBスキーマ自体（インデックス追加や正規化の見直しなど）には手を付けていない。テーブルが育つほど劣化する可能性が実測から示唆されているため、これは今後の課題として残す。

## 教訓

- AIに丸投げしたコードは、動いているように見えても「並列化されているか」程度の表面的なチェックだけでは不十分
- フロントエンドの実装が妥当でも、DevToolsのNetworkタブで実測しない限りボトルネックの所在は特定できない
- 「並列処理にしたから速いはず」という思い込みは、実測前の仮説に過ぎない
