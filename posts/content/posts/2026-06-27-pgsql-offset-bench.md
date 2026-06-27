---
title: "OFFSETページネーションをそのまま使ってませんか？――PostgreSQLで速度を実測してみた"
date: 2026-06-27T20:00:00+09:00
draft: false
ai_assisted: true
tags: ["postgresql", "database", "pagination", "performance", "sql", "keyset"]
categories: ["tech"]
description: "PostgreSQLでOFFSETページネーションとキーセットページネーションの速度を実測比較した。90万件目のページでは4000倍以上の差が出た。なぜOFFSETが遅いのか、内部動作から解説する。"
---

記事中で出てくる計測等は以下のリポジトリで実施した。

[GitHub - wasuken/pg-offset-bench · GitHub](https://github.com/wasuken/pg-offset-bench)

## はじめに

lobste.rsで "[All you need is PostgreSQL](https://ebellani.github.io/blog/2026/all-you-need-is-postgresql/)" という記事を読んだ。

「Redisやイベントストア、マイクロサービスに安直に逃げる前に、PostgreSQL単体で解決できないか考えろ」という主張を、金融システムをサンプルに証明する内容だった。

詳細については元記事を読んでくれ。

その中でキーセットページネーションに触れていて、私もOFFSETを使っていたので本当にまずいのか気になり、実際に環境を作って計測してみた。

ただし先に結論を言っておく。OFFSETが遅いのは事実だが、**特定のページ番号に直接ジャンプする処理が必須な場合はOFFSETを使うしかない**。キーセットは「前のページの続き」しか取れないため、任意のページへのランダムアクセスには対応できない。

逆に言えば、**次へ,前へや無限スクロールで十分なケースであれば、間違いなくキーセットページネーションを使うべきだ**。本記事はその判断材料として読んでほしい。

---

## 環境

- PostgreSQL 17（Docker）
- WSL2 / ArchLinux

```yaml
# compose.yml
services:
  postgres:
    image: postgres:17
    container_name: pg-offset-bench
    environment:
      POSTGRES_USER: bench
      POSTGRES_PASSWORD: bench
      POSTGRES_DB: bench
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init:/docker-entrypoint-initdb.d
    command: >
      postgres
        -c shared_buffers=256MB
        -c work_mem=16MB

volumes:
  pgdata:
```

テーブルは100万件のarticlesテーブルを用意した。中身はどうでもいいので適当。

```sql
CREATE TABLE articles (
    id         BIGSERIAL PRIMARY KEY,
    title      TEXT NOT NULL,
    body       TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

INSERT INTO articles (title, body, created_at)
SELECT
    'Article ' || i,
    repeat('body text ', 10),
    now() - (random() * interval '365 days')
FROM generate_series(1, 1000000) AS i;

CREATE INDEX idx_articles_created_at ON articles (created_at DESC, id DESC);

ANALYZE articles;
```

---

## OFFSETページネーションとは

もっとも一般的なページネーション実装。

```sql
-- ページ1
SELECT id, title, created_at
FROM articles
ORDER BY created_at DESC, id DESC
LIMIT 10 OFFSET 0;

-- ページ2
SELECT id, title, created_at
FROM articles
ORDER BY created_at DESC, id DESC
LIMIT 10 OFFSET 10;

-- ページ1001
SELECT id, title, created_at
FROM articles
ORDER BY created_at DESC, id DESC
LIMIT 10 OFFSET 10000;
```

直感的でわかりやすい。ORMもほぼすべてデフォルトでこれを使う。

---

## なぜOFFSETは遅いのか

`OFFSET 10000 LIMIT 10` というクエリを投げたとき、PostgreSQLが内部でやっていることはこうだ。

1. 先頭から10010行を読む
2. 最初の10000行を**捨てる**
3. 残り10行を返す

**スキップ**ではなく**読んで捨てる**。

`EXPLAIN ANALYZE` で実際に確認する。まず先頭（OFFSET 0）。

```
Limit  (cost=0.42..1.55 rows=10 width=30) (actual time=0.022..0.044 rows=10 loops=1)
  ->  Index Scan using idx_articles_created_at on articles  (cost=0.42..112027.85 rows=1000000 width=30) (actual time=0.021..0.042 rows=10 loops=1)
Planning Time: 0.344 ms
Execution Time: 0.073 ms
```

次に10万件目（OFFSET 100000）。

```
Limit  (cost=11203.17..11204.29 rows=10 width=30) (actual time=71.841..71.847 rows=10 loops=1)
  ->  Index Scan using idx_articles_created_at on articles  (cost=0.42..112027.85 rows=1000000 width=30) (actual time=0.017..69.288 rows=100010 loops=1)
Planning Time: 0.297 ms
Execution Time: 71.876 ms
```

`actual ... rows=` の値に注目してほしい。

- OFFSET 0 → `rows=10`：10行読んで10行返す
- OFFSET 100000 → `rows=100010`：**10万行読んで**10行返す

返しているのはどちらも10行だけなのに、読む量がまるで違う。

さらに問題がある。**INSERT/DELETEによるズレ**だ。

```
ページ1取得:  [A, B, C, D, E, F, G, H, I, J]  ← 10件返す
↓ 新規INSERTでZが先頭に入る
ページ2取得:  OFFSET 10 → [J, K, L, ...]  ← Jが重複する
```

逆にDELETEが起きるとレコードが欠落する。ORMを使っていると気づきにくい。

---

## キーセットページネーション

**N行目から**ではなく**この値より小さいものを**という発想に切り替える。

```sql
-- 前ページの最後のレコードの値をカーソルとして渡す
SELECT id, title, created_at
FROM articles
WHERE (created_at, id) < ('2024-06-01 12:00:00+09', 500000)
ORDER BY created_at DESC, id DESC
LIMIT 10;
```

`(created_at, id)` のタプル比較になっているのは、`created_at` が同じ値のレコードが複数あったとき `id` でタイブレークするためだ。

`EXPLAIN ANALYZE` で確認する。先頭相当（KEYSET offset~0）。

```
Limit  (cost=0.42..1.57 rows=10 width=30) (actual time=0.021..0.035 rows=10 loops=1)
  ->  Index Scan using idx_articles_created_at on articles  (cost=0.42..114527.83 rows=999999 width=30) (actual time=0.020..0.034 rows=10 loops=1)
        Index Cond: (ROW(created_at, id) < ROW('2026-06-27 10:21:17.689774+00'::timestamp with time zone, 998809))
Planning Time: 0.345 ms
Execution Time: 0.075 ms
```

10万件目相当（KEYSET offset~100000）。

```
Limit  (cost=0.42..1.66 rows=10 width=30) (actual time=0.056..0.185 rows=10 loops=1)
  ->  Index Scan using idx_articles_created_at on articles  (cost=0.42..111193.24 rows=898595 width=30) (actual time=0.055..0.183 rows=10 loops=1)
        Index Cond: (ROW(created_at, id) < ROW('2026-05-21 23:09:37.876801+00'::timestamp with time zone, 491583))
Planning Time: 0.385 ms
Execution Time: 0.233 ms
```

OFFSETと違い、`rows=` は先頭も10万件目も `10` のまま。インデックスを使ってカーソル位置に直接ジャンプしているため、ページの深さがコストに影響しない。

---

## アプリでの実装イメージ

ここでSQLを2回叩いてない？と思うかもしれない。

bench.shではカーソル位置の特定にOFFSETを1回使っているが、それはベンチ用の便宜的なコードだ。実際のアプリでは不要で、前のページを返すときにカーソル値をレスポンスに含めておくだけでいい。

```
クライアント              サーバー                DB
     |                       |                    |
     |-- GET /articles ------>|                    |
     |                       |-- SELECT LIMIT 10 ->|
     |                       |<-- rows + cursor ---|
     |<-- {data, cursor} ----|                    |
     |                       |                    |
     |-- GET /articles ------>|                    |
     |   ?cursor=xxx         |                    |
     |                       |-- WHERE (created_at, id) < cursor -->|
     |                       |<-- rows + next_cursor ---------------|
     |<-- {data, cursor} ----|                    |
```

レスポンス例：

```json
{
  "data": [...],
  "next_cursor": {
    "created_at": "2024-06-01T12:00:00+09:00",
    "id": 500000
  }
}
```

次ページリクエスト時にこのカーソルをそのままWHERE句に渡す。SQLは1発で完結する。

---

## 計測結果

同じページ位置でOFFSETとキーセットを比較した。

| ページ位置 | OFFSET | キーセット | 倍率 |
|---|---|---|---|
| 先頭（0件目） | 0.073ms | 0.075ms | 1x |
| 1,000件目 | 1.397ms | 0.118ms | **12x** |
| 10,000件目 | 14.512ms | 0.152ms | **95x** |
| 100,000件目 | 71.876ms | 0.233ms | **309x** |
| 500,000件目 | 279.035ms | 0.195ms | **1,431x** |
| 900,000件目 | 659.661ms | 0.164ms | **4,022x** |

OFFSETは線形に悪化する。100万件のテーブルで90万件目のページを取得すると659msかかる。キーセットは0.164msで変わらない。

---

## まとめ

OFFSETが遅い理由は2つ。

- **スキップではなく読んで捨てる** ——ページが深くなるほどコストが線形に増加する
- **INSERT/DELETEでズレる** ——重複・欠落が起きる。静かに壊れるので気づきにくい

ORMが黙ってOFFSETを使うので、意識しないと全員踏む。データ量が少ないうちは問題にならないが、数十万件を超えたあたりから効いてくる。

キーセットには制限もある。「任意のページ番号に飛ぶ」という操作ができない。ただ、それが必要なUIを本当に作っているかどうかは一度考え直す価値がある。無限スクロールや「次へ」ボタンであればキーセットで十分だ。

---

## 参考

- [All you need is PostgreSQL - Eduardo Bellani](https://ebellani.github.io/blog/2026/all-you-need-is-postgresql/)
- [Paging Through Results - Use The Index, Luke](https://use-the-index-luke.com/sql/partial-results/fetch-next-page)
- [No Offset - Use The Index, Luke](https://use-the-index-luke.com/no-offset)
- [Keyset Cursors, Not Offsets, for Postgres Pagination - Sequin](https://blog.sequinstream.com/keyset-cursors-not-offsets-for-postgres-pagination/)
- [Why You Should Avoid LIMIT OFFSET for Pagination in PostgreSQL - Eagle Eye](https://eagleeye.com/blog/why-you-should-avoid-limit-offset-for-pagination-in-postgresql)
