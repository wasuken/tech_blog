---
title: "テスト前提で設計したWebアプリのハンズオン - 読書管理アプリ その5"
date: 2026-03-05T01:30:00+09:00
tags: ["Next.js", "TypeScript", "React", "フロントエンド"]
categories: ["開発"]
---

## はじめに

[Part4](./part4) でカスタム例外クラスと残りのエンドポイントを実装し、APIが完成した。

Part5では `src/app/page.tsx` に簡易UIを実装する。バックエンドのロジックやテストには一切触れない。
フロントエンドからAPIを叩いて動くものを作るだけだ。

---

## 実装方針

`page.tsx` をClient Componentにする。

Server Componentでfetchする方法もあるが、ボタン操作のたびにstateを更新して再描画する必要がある。
今回のUIは「ボタンを押す → APIを叩く → 一覧を再取得して画面を更新する」という流れが中心なので、
`useState` + `useEffect` で管理するClient Componentのほうがシンプルだ。

```typescript
"use client";
```

ファイル先頭にこの1行を追加する。

---

## 実装するUI

- 本の追加フォーム（id / title / isbn）
- 本棚（Unread / Reading / Completed のグループ表示）
- 各本へのアクションボタン
  - Unread → 「読み始める」ボタン
  - Reading → 評価入力（1〜5）＋「読了にする」ボタン
  - 全ステータス → 削除ボタン

---

## 型定義

APIレスポンスの型を定義する。

```typescript
type BookStatus = "Unread" | "Reading" | "Completed";

type Book = {
  id: string;
  title: string;
  isbn: string;
  status: BookStatus;
  rating: number | null;
};
```

バックエンドの `ReadingStatus` は `as const` で定義した文字列リテラルなので、そのまま使える。

---

## データ取得

```typescript
async function fetchBooks() {
  try {
    const res = await fetch("/api/books");
    if (!res.ok) throw new Error("取得失敗");
    const data: Book[] = await res.json();
    setBooks(data);
  } catch (e) {
    setError(e instanceof Error ? e.message : "不明なエラー");
  } finally {
    setLoading(false);
  }
}

useEffect(() => {
  fetchBooks();
}, []);
```

`fetchBooks` は追加・ステータス変更・削除の後にも呼ぶ。
サーバーの状態を正として再取得するシンプルな方針だ。
楽観的更新（Optimistic Update）は今回やらない。

---

## 本の追加

```typescript
async function handleAdd(e: React.FormEvent) {
  e.preventDefault();
  setFormError(null);
  setSubmitting(true);
  try {
    const res = await fetch("/api/books", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(form),
    });
    const data = await res.json();
    if (!res.ok) throw new Error(data.error ?? "追加失敗");
    setForm({ id: "", title: "", isbn: "" });
    await fetchBooks();
  } catch (e) {
    setFormError(e instanceof Error ? e.message : "不明なエラー");
  } finally {
    setSubmitting(false);
  }
}
```

ISBNが不正な場合など、APIが400を返したときはフォームの下にエラーメッセージを出す。

---

## 読了時の評価入力

`ratingInputs` という `Record<string, string>` で各本のrating入力値を管理する。

```typescript
const [ratingInputs, setRatingInputs] = useState<Record<string, string>>({});
```

bookIdをkeyにして、それぞれの入力値を独立して持つ。
複数の本が同時に「Reading」状態でも干渉しない。

```typescript
async function handleComplete(id: string) {
  const ratingRaw = ratingInputs[id];
  const rating = ratingRaw ? parseInt(ratingRaw, 10) : undefined;
  const body = rating !== undefined ? { rating } : {};
  const res = await fetch(`/api/books/${id}/complete`, {
    method: "PATCH",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(body),
  });
  if (res.ok) {
    setRatingInputs((prev) => {
      const next = { ...prev };
      delete next[id];
      return next;
    });
    await fetchBooks();
  }
}
```

ratingは省略可能なので `parseInt` した結果が `undefined` のときは空のbodyを送る。
これはPart4で `req.json().catch(() => ({}))` として空ボディを許容している設計と対になっている。

---

## グループ表示

ステータスごとに本をグループ化して表示する。

```typescript
const grouped: Record<BookStatus, Book[]> = {
  Unread: books.filter((b) => b.status === "Unread"),
  Reading: books.filter((b) => b.status === "Reading"),
  Completed: books.filter((b) => b.status === "Completed"),
};
```

空のグループは表示しない。

```typescript
{(["Unread", "Reading", "Completed"] as BookStatus[]).map((status) => {
  const group = grouped[status];
  if (group.length === 0) return null;
  return (/* ... */);
})}
```

---

## ディレクトリ構成の確認

Part5での変更は1ファイルだけ。

```
src/
  app/
    page.tsx   # ← Client Componentに書き換え
```

---

## 動作確認

```bash
npm run dev
```

ブラウザで `http://localhost:3000` を開く。

1. フォームに id / title / isbn を入力して「追加する」
2. 追加した本が「積読」グループに表示される
3. 「読み始める」ボタンで「読中」に移動する
4. 評価を入力して「読了にする」ボタンで「読了」に移動する
5. ✕ボタンで削除できる

ISBNを12桁にして追加しようとするとAPIが400を返し、フォーム下にエラーが表示されることも確認しておく。

テストは変更なし。

```bash
npm run ci
# 15テスト、全パス
```

---

## 所感：フロントエンドにロジックを書かない

今回のUIはAPIを叩くだけで、ビジネスロジックを一切持っていない。

「積読→読了の直接遷移は不可」というルールはドメイン層の `ReadingStatusPolicy` が持っている。
フロントエンドでボタンの出し分けはしているが、それはUX上の都合であってバリデーションではない。
ボタンがなくても直接curlで叩けばAPIが422を返す。

ロジックの重複をなくすことで、将来モバイルアプリを作っても同じルールが適用される。

---

## まとめ

Part5でやったこと：

- `page.tsx` をClient Componentとして実装した
- 本の追加フォーム・ステータス変更・削除をUIから操作できるようにした
- 読了時のrating入力を各本ごとに独立したstateで管理した
- フロントエンドにビジネスロジックを持たせない方針を維持した
- テストは15件のまま全パス（フロントはE2Eで担保する方針）

次はUser/ReviewのDB拡張とReviewServiceの実装を予定。


# ここまで実装(生成)してわかったこと

アプリの安定感が違う。

これまでの趣味のアプリ開発はある程度要望だけ書いてみてくれというだけだったが、

コア部分を単体テスト書いた・・・というより、**テストするために最適化**したために

責任がより詳細に、それでいて具体的に可視化したような・・・気がする。

しかし、Claudeがすごかっただけかもしれないので気のせいかもしれない。
