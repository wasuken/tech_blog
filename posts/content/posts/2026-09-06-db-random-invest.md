---
title: "PostgreSQLのrandom()、SELECTとORDER BYで同じ式書くと使い回される"
date: 2026-10-18T14:00:00+09:00
draft: false
ai_assisted: true
tags: ["PostgreSQL", "MySQL", "SQLite", "SQL", "docker-compose"]
categories: ["技術"]
description: "タロットアプリで踏んだPostgreSQLのrandom()使い回しバグの原因調査と、MySQL/SQLiteとの挙動比較検証のメモ"
---

# 担当

- タイトル案、本文全般：私
- 環境・サンプルコード系全般：Claude
- SQL検証解説：Claude
- ヘッダ生成

# TL;DR

暇つぶしに書いた[タロットカードアプリ](https://github.com/wasuken/arcana-card)で踏んだバグの話。
PostgreSQLの仕様の読み間違えによるバグが思ったより分かりづらかったのでその流れと判明した具体的な原因、そしてそれらにまつわる検証内容を記事にする。

アプリはdocker compose環境下で動くようになっている。

# 時系列

## タロットカードの結果が4回連続逆位置・・・気の所為か

タロットカードアプリを書いた。コード自体はほぼCodexで生成した。細部の実装までは見てない。

実際に使ってみて4日、逆方向が妙に続いている。

確率としては 1/2^4 = 1/16 = 0.0625（6.25%）

約6%程度の可能性があるので、まあ・・・気の所為レベルではある。

とはいえ、少し気になったのでCodexに「逆位置が妙に続いているので実装に問題ないか調べてほしい」と指示→結論だけいえば「問題ありません」というニュアンスの解答が戻ってきたので安心（？）して閉じた。

## 5回目も逆方向

でも実装に問題ないと言われてるし、偶然か

## 6回目も逆方向は流石におかしい

というわけで再度Codexに調査を依頼。

とはいえ、抽象的な指示だと同じような結果になることはわかったので

実際にバックアップを取得した上で、実際に無限にタロットを引けるようにして検証コード作ってもらって検証して調査するように指示した。

結果、以下のSQLに原因があった。

```sql
SELECT id, number, name, upright_meaning, reversed_meaning,
       random() < 0.5 AS is_reversed
FROM tarot_cards
ORDER BY random()
LIMIT 1;
```

このSQLには二箇所random()が存在するが、
PostgreSQLで実行するとなんとrandom()の結果が使い回されてしまうようだ。

今回の実装では大アルカナのみなのでカードは22枚用意した。

ORDER BY random() LIMIT 1は、乱数が最も小さいカードを選ぶため22個の乱数から最小値を選んだ場合、その値が0.5以上になるのは"22個すべてが0.5以上"の場合だけであるため
確率は

(1/2)^22 = 1 / 4,194,304

となる。

つまり元の実装では

* 逆位置：約99.999976%
* 正位置：約0.000024%

となる。

そりゃ全部逆になるわけだ。

## 修正SQL

```sql
WITH picked_card AS MATERIALIZED (
    SELECT id, number, name, upright_meaning, reversed_meaning
    FROM tarot_cards
    ORDER BY random()
    LIMIT 1
), picked AS MATERIALIZED (
    SELECT picked_card.*, random() < 0.5 AS is_reversed
    FROM picked_card
)
SELECT id, number, name,
    CASE WHEN is_reversed THEN 'reversed' ELSE 'upright' END,
    CASE WHEN is_reversed THEN reversed_meaning ELSE upright_meaning END
FROM picked;
```

素直にWITHで区切ったところ、期待通りにタロットカードの位置がバラついた。

# 他のDBはどうなのか

MySQL, PostgreSQL, SQLite あたりの有名どころではどのような挙動を取るのか気になったので実際に確認してみた。

なお、微調整はしつつも基本コードや環境はすべてClaudeによって生成した。

また、PostgreSQLのバージョンについては一旦最新の18のイメージを利用した。

アプリと同じ16でもよかったが、興味を優先したので一旦最新で検証することにした。

## コード

検証コードだが、最初は各種SQLで実行しようと考えたが、あちこちマウントしたりコピーするのがだるかったのでPythonスクリプト埋め込み方式に変更した。

```
random_invest/
├── compose.yml
├── Dockerfile
└── test.py
```

postgres/mysqlコンテナは外部ポートを公開せず、runnerコンテナから内部ネットワーク経由で直接接続する構成にした。

なお検証用のSQLは、アプリ側の修正版（WITHで区切った形）ではなく、あえて元の壊れた1本のSELECT文の形に寄せている。バグを再現・証明するのが目的なので、WITHで区切って直ってしまった形で検証しても意味がないため。

## PostgreSQL

検証コード:

```sql
CREATE SEQUENCE call_counter;

SELECT id, nextval('call_counter') < 3 AS flag
FROM t
ORDER BY nextval('call_counter');

SELECT last_value FROM call_counter;
```

結果:

```
rows: [(1, True), (2, True), (3, False), (4, False), (5, False)]
times_called: 5
```

実行計画:

```
Result
  Output: id, ((random()) < '0.5'::double precision), (random())
  ->  Sort
        Output: id, (random())
        Sort Key: (random())
        ->  Seq Scan on public.t
              Output: id, random()
```

**解説:** PostgreSQLはターゲットリストの式を再利用するため、5行に対して呼び出し回数は5回。`EXPLAIN VERBOSE`を見ると、`Seq Scan`の段階で計算された`random()`の値がそのまま`Sort Key`にも使い回されているのが分かる。これが今回のバグの直接的な証拠。

## MySQL

検証コード:

```sql
SET @cnt := 0;

SELECT id, (@cnt := @cnt + 1) < 3 AS flag
FROM t
ORDER BY (@cnt := @cnt + 1);

SELECT @cnt;
```

結果:

```
rows: [(1, 1), (2, 0), (3, 0), (4, 0), (5, 0)]
times_called: 10
```

本物のRAND()での実演:

```sql
SELECT id, RAND() < 0.5 AS flag
FROM t
ORDER BY RAND();
```

```
rows: [(4, 1), (1, 0), (2, 1), (5, 0), (3, 1)]
```

**解説:** MySQLは公式ドキュメントに「ORDER BYやGROUP BYで揮発性関数を使うと複数回評価されうる」と明記されている通り、呼び出し回数は10（5行×2回）。独立評価されるため、PostgreSQLのようなバイアスは発生しない。ただし`EXPLAIN`は`Sort: rand()`としか出さず、式が使い回されているかどうかまでは分からない。

## SQLite

検証コード:

```python
def next_val():
    counter["n"] += 1
    return counter["n"]

conn.create_function("next_val", 0, next_val, deterministic=False)

rows = conn.execute("""
    SELECT id, next_val() < 3 AS flag
    FROM t
    ORDER BY next_val()
""").fetchall()
```

結果:

```
rows: [(1, 1), (2, 0), (3, 0), (4, 0), (5, 0)]
times_called: 10
```

実行計画:

```
(3, 0, 0, 'SCAN t')
(14, 0, 0, 'USE TEMP B-TREE FOR ORDER BY')
```

**解説:** SQLiteは公式フォーラムで「エンジンの効率判断次第で同じ式でも複数回評価されることがある」と説明されており、仕様として保証されていない。今回試した環境では10（独立評価側の挙動）だったが、バージョンや実行計画次第で変わりうる。

## まとめ

| DB | 呼び出し回数 | 挙動 |
|---|---|---|
| PostgreSQL | 5 | 使い回し（1回だけ評価） |
| MySQL | 10 | 独立評価（仕様通り） |
| SQLite | 10 | 独立評価だが未規定・実装依存 |

# 結論

randomには気をつけよう。

未熟者が生成AIに実装を委ねたリスクなので素直に受け入れる。
