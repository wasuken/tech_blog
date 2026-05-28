---
title: "Gemini API で財務書類を「怪しさ判定」する：スコア付き出力の設計"
date: 2026-05-24T00:00:00+09:00
draft: false
tags: ["Gemini", "AI", "TypeScript", "プロンプト設計", "個人開発"]
categories: ["個人開発"]
description: "Gemini API に有価証券報告書の PDF を渡して財務分析させる。normal / caution / danger のスコアをプロンプトで定義し、出力をパースして DB に保存する設計を解説する。"
---

個人開発の EDINET 分析ツールでは、取得した有価証券報告書の PDF を Gemini に渡して「怪しさ判定」をさせている。  
単なる要約ではなく、**3段階のスコア（normal / caution / danger）** を返させる設計にしたので、その仕組みをまとめる。

## なぜスコアが必要か

毎日数十〜数百件の書類が提出される。全部読むのは無理なので、AI に「これは要注意」かどうかを仕分けさせたい。  
スコアが `danger` の書類だけ Discord 通知を飛ばす、といった使い方ができる。

## プロンプト設計

プロンプトの末尾に必ずスコアを出力させるよう指示する：

```
分析の最後に必ず以下の形式でスコアを出力してください：

SCORE:normal   # 特に問題なし
SCORE:caution  # 気になる点あり・要確認
SCORE:danger   # 重大なリスクの可能性
```

Gemini はマークダウン形式で分析テキストを返した後、最終行に `SCORE:danger` のような文字列を出力する。

## PDF を渡す方法

`@google/generative-ai` SDK では PDF を base64 で渡せる：

```typescript
const model = genAI.getGenerativeModel({ model: 'gemini-2.5-flash' })

const result = await model.generateContent([
  {
    inlineData: {
      mimeType: 'application/pdf',
      data: pdfBuffer.toString('base64'),
    },
  },
  { text: prompt },
])
```

最大 50MB まで渡せるが、大きすぎるとトークン消費が跳ね上がるので注意。

## スコアのパース

正規表現で `SCORE:` 以降を抽出：

```typescript
function parseScore(text: string): 'normal' | 'caution' | 'danger' {
  const match = text.match(/SCORE:(normal|caution|danger)/i)
  return (match?.[1] ?? 'normal') as 'normal' | 'caution' | 'danger'
}
```

この1行で取れる。テキスト全体で最初にマッチした `SCORE:xxx` を使うので、**プロンプトの中に `SCORE:` を書いてしまうと誤マッチする**。プロンプト側では `SCORE:normal` という形式を「例として」書かずに指示だけにするか、出力セクションを明示的に区切るとよい。

## DB への保存

`analysis_results` テーブルの `score` カラムに `CHECK` 制約を入れている：

```sql
score VARCHAR(10) CHECK (score IN ('normal', 'caution', 'danger'))
```

これで不正な値が入らない。プロンプトの出力がブレてもDB側で弾ける。

## まとめ

- プロンプト末尾に `SCORE:xxx` を出力させる設計はシンプルで使いやすい
- PDF は base64 で `inlineData` として渡す
- スコアのパースは正規表現1行
- DB の `CHECK` 制約でバリデーションを二重にかける
