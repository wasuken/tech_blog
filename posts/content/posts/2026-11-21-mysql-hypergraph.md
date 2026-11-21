---
title: "MySQL 9.7 の Hypergraph を ON/OFF で比べた"
date: 2026-11-21T12:00:00+09:00
draft: false
ai_assisted: true
categories: ["雑テック"]
tags: ["MySQL", "MySQL 9.7", "Hypergraph", "オプティマイザ", "Sakila", "Docker"]
description: "MySQL 9.7 の Hypergraph オプティマイザを ON にすると、JOINの多いクエリは3〜5倍速くなったが、等価条件のない範囲JOINだけ10〜28倍遅くなった。Docker + Sakila の小規模な検証メモ。"
---

## TL;DR

Docker上の Sakila で見る限り、Hypergraph ON は多表JOINで3〜5倍速く、等価条件のない範囲JOINで10〜28倍遅くなった。

- 速くなった: j05 / j08 / j12 / lo(約 0.15〜0.41 倍の時間)
- 遅くなった: 範囲JOINだけ(約10〜28倍)。原因は CTE ではなく、等価条件がないこと
- 対処: その文だけ `SET_VAR` で OFF、または等価条件に書き換え

これは趣味の小さな検証で、根拠は薄い。原因(なぜコストモデルが選ぶか)は未確認で、Oracle への報告もしていない。

## 背景

MySQL 9.7 LTS(2026-04-21 リリース)で、Hypergraph オプティマイザが Community Edition で使えるようになった。既定では OFF。

- 有効化: `SET [SESSION] optimizer_switch='hypergraph_optimizer=on'`、または文単位の `/*+ SET_VAR(optimizer_switch='hypergraph_optimizer=on') */`
- Oracle 自身が、全ワークロードで最適とは限らず、TPC-DS SF1 で大きな悪化例があると書いている
- 悪化を見つけたら、再現手順・スキーマ・索引・EXPLAIN ANALYZE を付けてバグ報告してほしいという立場

この検証の動機は、「常時 ON でよさそう」と思ったが、手元のクエリでそれを確かめていなかったこと。公式データセット Sakila での、小さなサンプルとしての比較。

## 環境と測り方

**環境**

- Docker コンテナ: mysql84 (3306) = MySQL 8.4.13 LTS、mysql97 (3307) = MySQL 9.7.4 LTS
- データ: 公式 Sakila(基本テーブル16、rental 約16,044行)
- 3構成: 8.4 / 9.7 Hypergraph OFF(97off) / 9.7 Hypergraph ON(97on)

**測定手順 (compare.sh)**

1. 事前に全テーブルを `ANALYZE TABLE`
2. `EXPLAIN ANALYZE` の最上位ノードの `actual time=A..B` の B(ms)を採用
3. ウォームアップ1回の後、RUNS(3、一部は5)回の中央値
4. 97off と 97on は交互に実行し、毎回 `SET SESSION optimizer_switch` を切り替える
5. 結果の正しさ: 結果行をソートして md5 を取り、3構成で一致するか見る
6. プランが分割されて EXPLAIN ANALYZE で測れないクエリだけ、元の SELECT のサーバ側実行時間(performance_schema の `TIMER_WAIT`、`ps_ms`)を10回測って補う

**判定の定義**

- plan_changed: プランからコストと actual time の括弧を除いて木を比べたもの
- hg_used: optimizer_trace に "Constructed hypergraph" があるか
- WORSE: 比率 > 1.2 かつ plan_changed=Y。プランが同じなら「差はノイズの可能性」と注記
- n/a(split): 出力に "printed elsewhere" を含むとき(CTE を複数回参照してプランが分割される)。この場合は `ps_ms` の中央値で判定する

## クエリ8本

| 名前 | 狙い |
| --- | --- |
| j05 | 5テーブルの JOIN |
| j08 | j05 + customer / address / city |
| j12 | 12テーブルの INNER JOIN |
| lo | LEFT JOIN + NOT EXISTS |
| cte1 | CTE を1回参照 |
| cte2 | CTE を2回参照(プランが分割される) |
| cter | 再帰CTE(120日分の日付) + 範囲JOIN |
| s01 | 対照群: film + language の集計 |

範囲JOIN の中身(cter)は次のとおり。

```sql
WITH RECURSIVE rental_days (rental_day, day_number) AS (
    SELECT DATE('2005-05-01'), 1
    UNION ALL
    SELECT rental_day + INTERVAL 1 DAY, day_number + 1
    FROM rental_days WHERE day_number < 120
)
SELECT d.rental_day, COUNT(r.rental_id) AS cnt
FROM rental_days d
LEFT JOIN rental r ON r.rental_date >= d.rental_day
                  AND r.rental_date < d.rental_day + INTERVAL 1 DAY
GROUP BY d.rental_day ORDER BY d.rental_day;
```

## 結果1: 本表

単位 ms、中央値。測定方法を直したあとの compare.sh の値。

| クエリ | 8.4 | 9.7 OFF | 9.7 ON | ON/OFF |
| --- | --- | --- | --- | --- |
| cte1 | 33.6 | 29.9 | 36.6 | 1.224 |
| cte2 | n/a(split) | n/a(split) | n/a(split) | 下の補足を参照 |
| cter | 16.3 | 35.2 | 335 | 9.517 |
| j05 | 45.7 | 49.8 | 12.8 | 0.257 |
| j08 | 83.3 | 114 | 24.3 | 0.213 |
| j12 | 172 | 179 | 42.8 | 0.239 |
| lo | 42.5 | 41.4 | 17.1 | 0.413 |
| s01 | 2.96 | 2.16 | 1.74 | 0.806 |

読み方:

- j05 / j08 / j12 / lo: ON が OFF の約0.2〜0.4倍。一貫して速い
- cter: ON が約10倍遅い。これだけ大きく悪化する。測定修正前の1回目は OFF が速く出て、比が16.8だった
- cte1: WORSE のフラグが立ったが +6.7 ms で、修正前の1回目は逆に少し速かった(0.955)。ノイズと判断
- cte2: プランが分割されて EXPLAIN ANALYZE では測れなかったため、サーバ側の実行時間 `ps_ms`(元の SELECT、10回)で測り直した。中央値は OFF 13.6 ms、ON 12.6 ms(比 0.93)。最小〜最大は OFF 8.7〜27.1 ms、ON 11.7〜26.5 ms で、差はぶれの範囲内。`ps_ms` は EXPLAIN ANALYZE とは別の測り方なので、表の他の行の数字とは直接比べない
- cter の 8.4 / OFF は回ごとにぶれる

## 結果2: 悪化の原因の切り分け

最初は「再帰CTEが悪い(行数見積りが外れる)」を疑った。結論から言うとこれは外れ。原因は範囲JOIN。

1. **cter のプラン差分**: OFF は Nested loop left join + `Index range scan on r (re-planned for each iteration)`。ON は `Left hash join (no condition)` で、rental をカバリング索引で全件スキャンしてハッシュ側にする。外側の行数 N × 約16,000 回の比較になる
2. **cter_nr(再帰をやめて、film から日付を作る)**: ON 542 / 512 / 522 / 501 ms、OFF 48.7 / 18.3 / 19.1 / 47.8 ms。再帰を外しても遅い。→ 再帰CTEは無実
3. **cter_nr_eq(等価に書き換え、`DATE(r.rental_date) = d.rental_day`)**: ON 29.3 / 26.7 ms。悪化しない
4. **cter_r(再帰CTE + 等価条件)**: ON 13.0 / 23.3 / 10.6 ms 対 OFF 10.7 / 21.9 / 11.6 ms。プラン同じ、差なし
5. **ヒント NO_BNL(r)**: ON で無視され、ハッシュジョインのまま 514 / 503 ms。プラン差分も無し
6. **文単位で OFF**: `/*+ SET_VAR(optimizer_switch='hypergraph_optimizer=off') */` で 20.3 / 20.7 / 20.1 ms。hg_used=N、プランは OFF と同一

ここから言えること: 「等価条件がなく、範囲で JOIN する」と ON で遅くなる。「なぜコストモデルがそれを選ぶか」は確認できていない。コストの見積り違いか、パラメータ付き範囲スキャンが候補にならないかの仮説だけ。

## 結果3: 外側の行数Nを変える実験

N を増やして、範囲JOINと、等価+索引JOIN(`customer c LEFT JOIN rental r ON r.customer_id = c.customer_id WHERE c.customer_id <= N`)を比べた。単位 ms、OFF → ON。

| N | 範囲JOIN (OFF → ON) | 等価+索引 (OFF → ON) |
| --- | --- | --- |
| 1 | 0.083 → 22.8 | 0.061 → 0.054 |
| 10 | 0.213 → 54.5 | 0.221 → 0.247 |
| 100 | - | 2.000 → 1.680 |
| 120 | 33.4 → 530 | - |
| 500 | 50.0 → 1393 | - |
| 599 | - | 14.1 → 15.2 |

- 範囲JOIN: ON はどの N でも `Left hash join (no condition)`。N=1 でも約22.8 ms かかり、N とともに増える(N=500 で約1.4秒)
- 等価+索引: ON もどの N でも `Nested loop left join`。ネステッドループを見逃すことはなかった
- 等価条件があれば、Hypergraph は索引付きネステッドループをちゃんと選んでいた

## 測定の落とし穴

- **cte2 の「555倍悪化」は偽物**: CTE を2回参照するとプランが `query plan printed elsewhere ... (never executed)` と分割され、最上位ノードの時間(0.055 ms)が当てにならない。壁時計の4回(クライアント起動込み)では ON ≈ OFF だったが精度が低いため、compare.sh にサーバ側計測(`ps_ms`)を足して10回で測り直した。結果は OFF 13.6 ms、ON 12.6 ms で差なし
- **OFF 自体がぶれる**: 同じクエリの OFF が実行ごとに2〜2.7倍動く(cter_r 10.7→21.9 ms、cter_nr 48.7→18.3→19.1→47.8 ms)。10〜30 ms の差はノイズ扱い
- **plan_changed は粗い**: 結合方式が同じ(eqidx)でも Y になる。「プランが変わった」と「結合方式が変わった」は別
- **WORSE の誤検知**: cte1(+6.7 ms、比1.224)は絶対差が小さくノイズ。絶対差のしきい値(例: 10 ms)を入れる案は未実施
- **ホストの `mysql` コマンドは実は MariaDB クライアント**: 「Deprecated program name ... use '/usr/bin/mariadb'」と出る。MySQL 9.7 には繋がったが、壁時計にはクライアントを変える影響があり得る

## 似た報告と Oracle の設計意図

**似たバグ報告**(中身は検索で見つけたもので、自分の環境での再現はしていない)

	- [#120870](https://bugs.mysql.com/bug.php?id=120870): 範囲 / BETWEEN が等価JOINとして抽出されず、条件なしハッシュジョインになる。今回の症状に一番近い
- [#120830](https://bugs.mysql.com/bug.php?id=120830)
- [#120809](https://bugs.mysql.com/bug.php?id=120809): 9.7.1、40M 行のハッシュジョインで直積、Verified
- [#120452](https://bugs.mysql.com/bug.php?id=120452): Hypergraph 改善のチケット、Verified

**Oracle の設計意図**: Oracle のブログでは、ネステッドループとハッシュジョインを同じコストモデルで比べるため、索引付きネステッドループが自動で勝つわけではない。今回の結果はその性質と矛盾しないが、これが設計どおりなのかバグなのかは判断できない。報告すると重複か「仕様」と返される可能性もある。

## 使い方の指針

この環境の結果から考える目安。一般化はしない。

1. まずセッションで ON にして、自分の重いクエリを `EXPLAIN ANALYZE` で OFF と比べる
2. 等価条件のない範囲JOIN(`>=` / `<` / BETWEEN で結ぶもの)を探す。プランに `Left hash join (no condition)` が出たら要注意
3. 悪化した文は `SET_VAR(optimizer_switch='hypergraph_optimizer=off')` で、その文だけ OFF に戻す
4. 書き換えできるなら等価条件にする。ただし `DATE(col) = x` のような変換は索引が効かなくなる点に注意
5. `NO_BNL` や HASH_JOIN 系のヒントは、この環境では ON 下で効かなかった

「常時 ON」にする前に、上の 1〜2 をやっておくのが無難。

## 限界と未確認のこと

- 反復は3〜5回(cte2 の `ps_ms` は10回)。同じクエリのOFFでも2〜2.7倍ぶれるので、小さい差は判断できない
- Sakila は小さい(rental 約16,000行)。大きいデータでの挙動は見ていない
- MySQL 9.7.4 のみ、Docker 1台、時間は EXPLAIN ANALYZE の値(cte2 のみサーバ側実行時間)
- コストモデルがなぜ条件なしハッシュジョインを選ぶかは未確認。コストの過小見積りか、パラメータ付き範囲スキャンが候補にならないかのどちらかだと疑っているだけ
- CTE の他のパターン(ウィンドウ関数など)は試していない
- 重複・仕様の可能性があるため、Oracle にはバグ報告していない

## 出典

- [The Hypergraph optimizer is now available in MySQL 9.7 Community Edition](https://blogs.oracle.com/mysql/the-hypergraph-optimizer-is-now-available-in-mysql-9-7-community-edition) (Oracle): 有効化方法と、全ワークロードで最適とは限らないという注意
- [Smarter join planning with the Hypergraph optimizer](https://blogs.oracle.com/mysql/smarter-join-planning-with-the-hypergraph-optimizer) (Oracle): 設計の説明
- [Optimizer Hints (9.7)](https://dev.mysql.com/doc/refman/9.7/en/optimizer-hints.html): SET_VAR / NO_BNL
- [Hash joins (9.7)](https://dev.mysql.com/doc/refman/9.7/en/hash-joins.html)
- [Switchable optimizations (9.7)](https://dev.mysql.com/doc/refman/9.7/en/switchable-optimizations.html)
- [Notes on MySQL 9.7 LTS release](https://www.bytebase.com/blog/notes-on-mysql-9-7-lts-release/) (Bytebase)
- バグ: [#120870](https://bugs.mysql.com/bug.php?id=120870) / [#120830](https://bugs.mysql.com/bug.php?id=120830) / [#120809](https://bugs.mysql.com/bug.php?id=120809) / [#120452](https://bugs.mysql.com/bug.php?id=120452)
- リポジトリ: [wasuken/mysql-8.4-vs-9.7](https://github.com/wasuken/mysql-8.4-vs-9.7)
