---
title: "ReactのuseStateでDate.now()を使うとlintエラーになる話"
date: 2026-05-23T00:00:00+09:00
draft: false
tags: ["React", "TypeScript", "hooks", "lint"]
categories: ["小ネタ"]
description: "useState の初期値で Date.now() を直接呼ぶと react-hooks/purity エラーになる。lazy initialization で解決する小ネタ。"
---

## 何が起きたか

`useState` の初期値で `Date.now()` を直接呼んでいたら、こんなエラーが出た。

```
error  Error: Cannot call impure function during render
`Date.now` is an impure function.
```

コード的にはこういうやつ。

```tsx
const [stockPriceFrom, setStockPriceFrom] = useState<string>(
  new Date(Date.now() - 90 * 86400000).toISOString().split('T')[0]
)
```

## なぜエラーになるか

React のルールとして、**レンダー中はコンポーネントが pure でなければならない**。

`Date.now()` や `Math.random()` はレンダーのたびに異なる値を返す「impure な関数」なので、直接渡すと React（特に Strict Mode）に怒られる。

Strict Mode ではレンダーを意図的に 2 回実行するため、こういった副作用が顕在化しやすい。

## 解決策：lazy initialization

`useState` に関数を渡すと、**初回マウント時に一度だけ実行**される。これが lazy initialization パターン。

```tsx
const [stockPriceFrom, setStockPriceFrom] = useState<string>(
  () => new Date(Date.now() - 90 * 86400000).toISOString().split('T')[0]
)
const [stockPriceTo, setStockPriceTo] = useState<string>(
  () => new Date(Date.now() - 84 * 86400000).toISOString().split('T')[0]
)
```

`() =>` で包むだけ。それだけ。

## まとめ

| パターン | 評価タイミング |
|---|---|
| `useState(Date.now())` | レンダーごとに評価される |
| `useState(() => Date.now())` | 初回マウント時のみ |

`new Date()` も同様なので、日付系の初期値を `useState` に渡すときはアロー関数で包む癖をつけておくと良い。

参考：[React 公式 — Components and Hooks must be pure](https://react.dev/reference/rules/components-and-hooks-must-be-pure#components-and-hooks-must-be-idempotent)
