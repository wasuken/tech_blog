---
title: "ARCHITECTURE.mdの見出しをAIに消されないために、小さいCLI「archmd」を作った"
date: 2026-11-14T11:00:00+09:00
draft: false
ai_assisted: true
tags: ["Go", "設計", "個人開発", "AI", "CLI"]
categories: ["技術"]
description: "ARCHITECTURE.md をエージェントに更新させる運用を、文書内に常駐するマーカーと検証で回す Go 製 CLI の雑メモ"
---

雑メモ。設計はClaudeと壁打ちし、実装はCodexに投げた。挙動はテストと実際の利用で確認している。

## TL;DR

- AIに「ARCHITECTURE.md を更新して」とだけ頼むと、**見出しが消える**。全体を書き直させているのが原因だと判断した
- ARCHITECTURE.md の中に、全12節ぶんの「消すな」マーカーを常駐させ、**ID で検証する** CLI を Go で作った → [wasuken/archmd](https://github.com/wasuken/archmd)
- CLI は LLM を呼ばない。骨組みの配置、検証、分割/結合、更新依頼文の出力だけを担当する。中身を書くのはエージェント

## 規模感

| 項目 | 値 |
|---|---|
| 言語 | Go 1.24 以上、標準ライブラリのみ |
| 行数 | 約 2,900 行(テスト込み) |
| テスト関数 | 61 個 |
| コミット | 16 個(2026-11-12 初版 → 11-14 に v0.3.6) |
| 版 | v0.3.6(git タグから取得) |
| コマンド | init / prompt / check / split / build / template / notices / version / help / remove |

## 動機

- [architecture.md](https://architecture.md/) のテンプレで ARCHITECTURE.md を書いていたが、運用が面倒だった
- AI に更新を頼むと、見出しの一部が消える
- 別の PJ では ARCHITECTURE.md が 500 行近くに膨らんでいた。分割も考えたが、原因は検証ログや手順が混ざっていたこと。**分割より先に削るべきだった**

## 仕組み:マーカーを文書に常駐させる

ARCHITECTURE.md には、既知の 12 節すべてのマーカー行が順番に入っている。マーカーは ID・見出し・何を書くかのヒントを持つ。

```markdown
<!-- arch:project-structure | Project layout | Map the main directories to their responsibilities. -->
## Project layout
(本文)

<!-- arch:high-level-diagram | System map | Show the system overview and connections between major components. -->
## System map
(本文)
```

ルールは 3 つだけ。

1. **マーカー行は絶対に消さない・編集しない**
2. 書くことが無い節は、見出しと本文だけ消す(マーカーは残る)
3. 空の節を埋めるときは、マーカー内の見出しで直下に H2 を書く

照合は見出しの文字列ではなく ID で行うので、見出しや本文を日本語にしても検証は通る。マーカー行は行数に数えない。

## 設計判断(ざっくり)

| 論点 | 決めたこと | なぜ |
|---|---|---|
| 見出しが消える | 全節のマーカーを文書内に常駐させ、ID で照合する | 消えたら機械で気づける |
| 「欠落はエラー」と「不要な節は消す」の矛盾 | 必須節は 4 つ(project-structure / high-level-diagram / core-components / project-identification)。不要な節は本文だけ消し、マーカーは残す | 小さい PJ に 12 節全部は要らない。でも節の存在は失いたくない |
| マーカーが消えたら | check が `WARN [missing-marker]` で、足すべきマーカー行と位置を示す | 直し方まで出せば、エージェントが自分で戻せる |
| CLI の責務 | LLM 呼び出し・ネットワーク接続なし | エージェントの呼び方の差をツールで背負わない |
| 言語指定 | テンプレに書かず、AGENTS.md と `~/.config/architecture-md/common.md` に書く | テンプレを言語ごとに分けたくない |
| 既存の AGENTS.md | 既定は触らず、貼り付け用ブロックを標準出力に出す。`--write-agents` を付けたときだけ追記 | 使っている AGENTS.md を上書きされたら最悪なので、セーフティを優先した |
| 分割 | `.architecture/` を置いた PJ だけ結合モード(`.architecture/` が正本、ARCHITECTURE.md は生成物) | AI は索引方式が楽、人間は 1 ページで通読したい。両立はオプションで |
| 更新のトリガー | 「構造・コンポーネント・データストア・外部連携・デプロイが変わったら、同じタスク内で更新して check」をブロックに明記 | 「作業前に読め」だけで、「いつ更新するか」が無かった |
| `remove` | 既定は予定の表示だけ。`--yes` で実行 | 一番危ないコマンドなので |
| 版の刻印 | 古ければ check が INFO を出すだけ。upgrade コマンドは作らない | 各 PJ で手を入れた文書を上書きすると事故る |
| 版の出どころ | ソースの定数をやめ、`runtime/debug.ReadBuildInfo()` で git タグ由来の版を読む。取れなければ `dev` | 版を上げるたびにソースを書き換える手間と、タグとのズレを無くす |

### テンプレの置き場所で 2 回揺れ戻した

| 回 | 変更 | 戻した理由 |
|---|---|---|
| 1 回目 | バイナリに埋め込む → リポジトリに `ARCHITECTURE.md.template` を置く | 埋め込みだと、archmd が無い環境でエージェントが詰まる |
| 2 回目(v0.2.0) | `.template` を置く → ARCHITECTURE.md 本体にマーカーとして埋め込む | 別ファイルと照合させるより、「消すな」と書いたマーカーを本体に残す方が、元の問題にまっすぐ効く |

結果として、**運用の本体はリポジトリ内のテキスト(AGENTS.md と ARCHITECTURE.md)だけで、CLI は無くても回る**形に落ち着いた。

### `--write-agents` の安全策

指定したときの挙動は以下。

1. 既存の AGENTS.md(無ければ CLAUDE.md)を読む
2. archmd のマーカーを先に検証する。片方だけ・入れ子などの壊れたマーカーがあれば、**何も書かずにエラー**
3. すでにブロックがあれば何もしない
4. 無ければ、既存のバイト列の後ろに追記するだけ

貼り忘れ対策として、check は AGENTS.md からの参照漏れも警告する。

### 移行コマンドは作らなかった

v0.1.0 → v0.2.0 はマーカー形式が変わる破壊的変更。それでも upgrade コマンドは作らず、作り直しで済ませた。

- v0.x の利用者は自分だけ(AGENTS.md のブロック更新も、同じく作り直しで済ませている)
- ARCHITECTURE.md はコードから作り直せる
- check が旧形式を `WARN [legacy-format]`、残った `.template` を `WARN [legacy-template]` で知らせる
- v0.3.0 からは `remove --yes` が旧 `ARCHITECTURE.md.template` も消す。最初は「手で消して」にしていたが、作り直しの 3 コマンドで掃除まで終わる方が筋がいいので、remove の対象に入れた

```sh
# 消す前に、Key decisions などコードから再生成できない記述は退避しておく
archmd remove --yes            # ARCHITECTURE.md と旧 .template をまとめて削除
archmd init --write-agents
archmd prompt | <agent>
```

## 使い方

```sh
go build -o archmd .            # README と同じ手順

archmd init                     # マーカー入りの ARCHITECTURE.md を配置(必須4節だけ見出し付き)
archmd prompt | claude -p && archmd check
archmd prompt | codex exec - && archmd check

# 変更の文脈はシェルで足す
{ archmd prompt; git diff --stat main; } | claude -p

archmd remove                   # 予定の表示だけ
archmd remove --yes             # 実行
```

- `archmd prompt` は更新依頼文を標準出力に出すだけ。中身は、依頼・編集モードの指示・ルール本文・現在の check の診断・終了条件
- `codex exec` 末尾の `-` は、標準入力をプロンプトとして読む指定([Daniel Vaughan の解説](https://codex.danielvaughan.com/2026/04/15/codex-exec-unix-pipelines-prompt-plus-stdin/))。書き込み権限のフラグはエージェントや版で違うので、各ドキュメントで確認する
- 終了コードは正常 0、check の ERROR かファイル操作の失敗で 1、引数の誤りで 2

## ライセンスの話

元ネタの [timajwilliams/architecture](https://github.com/timajwilliams/architecture) を参考にしたので、出典表記で少し調べた。

- 最初は「MIT だからリンクを書けばいい」と思っていた。MIT でも、著作権表示とライセンス本文の同梱が要る(私の整理で、法律の専門家の見解ではない)
- 調べると、上流の **LICENSE ファイルは Apache License 2.0** だった。一方、上流の README とサイトは MIT と書いていて、表記が食い違っていた。LICENSE の Copyright 行は雛形のまま
- 条件が重い Apache-2.0 側に合わせた
  - SPDX は `Apache-2.0`、`LICENSE` に全文、`NOTICE` に出典・参照コミット・変更点を記載
  - `archmd notices` で全文を出せる
  - README と NOTICE に「Web と README は MIT 表記だが、参照コミットの LICENSE 本文に従い Apache-2.0 として扱う」と明記
- 見出しは AI と相談して独自の名前に整理した(Project layout / System map / Components / Data stores / External services / Build and deploy / Key decisions / Security / Development and testing / Roadmap and known debt / About this project / Glossary)。元の 11 節に Key decisions を足して 12 節

## ハマりどころ・小ネタ

- `arch` は Linux の既存コマンド(アーキテクチャ表示)と被るので `archmd` にした
- `go:embed` の中身はビルド時に焼き込まれる。変えるには再ビルドが要る。ビルドせずに変えられるのは common.md だけ。一瞬「更新が回らない?」と焦ったが、更新は AGENTS.md 経由でエージェントがやるので問題なかった
- init の案内は標準エラーに出す。標準出力は AGENTS.md に貼るブロック専用にしている
- WSL では Windows 側と home が別なので、common.md も 2 つに分かれる
- PowerShell 経由のパイプだと標準入力が UTF-8 にならないことがある。bash 内で実行する
- 自分の ARCHITECTURE.md で試したら日付が抜けていた → About this project に最終確認日を入れた
- split → build で元のバイト列が戻らない問題があり、v0.2.0 の直後に直した。結合モードの check はバイト単位の一致を見るので、ここがずれると即 drift になる
- 版をビルド情報から取るのは、Go 1.24 以降の `go build` が VCS のタグやコミットから main モジュールの版を埋めてくれるから([Go 1.24 Release Notes](https://go.dev/doc/go1.24))。手元の Go 1.24.7 で、タグ付きのコミットを `go build -o archmd .` すると `archmd version` はタグどおりの版になった
  - タグの無いコミットだと擬似版になり、未コミットの変更があれば `+dirty` が付く。archmd は擬似版でも先頭の数値で比較する
  - 最初は `go.mod` の下限が Go 1.22 のままで、この仕組みと噛み合っていなかった。v0.3.2 で下限を 1.24 に上げた

## 版を上げるたびに自分で INFO を踏む問題

版をビルド情報から取るようにしたら、自分のリポジトリで妙なループに入った。

1. AGENTS.md のブロックには、生成したバイナリの版が刻まれる
2. ブロックはコミットより前に作るので、刻まれるのは「今の版」
3. その後にタグを打つと版が上がり、check が `INFO [outdated-block]` を出す
4. 直してタグを打つと、また版が上がる

**タグを打つたびに、自分の通知を自分で踏む。** v0.3.3 と v0.3.4 はこれで INFO が残った。

手順で解決した。`release.sh` を作って、リリースを一括にしている。

| 順番 | やること |
|---|---|
| 1 | `go vet` / `gofmt -l .` / `go test` |
| 2 | 最新タグのパッチを +1 した仮タグを打つ |
| 3 | クリーンな状態でビルドし、新しい版をバイナリに焼き込む |
| 4 | そのバイナリで AGENTS.md のブロックだけ作り直す(ブロック外の行は残す) |
| 5 | コミットして、タグをそのコミットに付け直す |
| 6 | 再ビルドして、版と check が `0 ERROR, 0 WARN, 0 INFO` か確認 |
| 7 | Y/n を聞いて main とタグを push |

途中で失敗したら仮タグは消す。ARCHITECTURE.md には触らないので、先頭のライセンス表示も消えない。v0.3.5 以降はこれで出していて、check は全部 0 のまま。

ただし、タグより先の main をビルドすると擬似版(先頭が次のパッチ)になるので、INFO はまた出る。根本的には「バイナリの版」ではなく「ブロックの中身が変わった版」で比べるべきで、ここは次の課題にしている。

## AI の使い方

- 各タスクのプロンプトに**完了の定義**を書き、`go test` で検証できる形にした

| 完了の定義の例 |
|---|
| init を 2 回実行しても差分 0 |
| 既存の AGENTS.md のバイト列が前後で同じ |
| prompt を 2 回実行して同じ出力 |
| 実行前後でファイルが変わらない(remove の既定動作など) |
| `go vet` と `gofmt -l .` が空 |

- 同じファイル(agents-block.md、init)を触る追加タスクは 1 回にまとめて投げる。バラで投げると、Codex が毎回 golden を全部更新し直す

# 所感

人に PJ を説明するとき、ARCHITECTURE.md をコピペするだけでだいたい伝わるようになった。小さいが成功だと思う。この記事の素材集めも、ARCHITECTURE.md を貼るところから始めた。

**AI に文書を任せるなら、全体を書き直させずに、壊れたら機械で気づける目印を本体に残しておく。**

## 参考

- [wasuken/archmd](https://github.com/wasuken/archmd)(本記事の CLI、作者: wasuken)
- [timajwilliams/architecture](https://github.com/timajwilliams/architecture)(元テンプレ、作者: timajwilliams)
- [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0.txt)(Apache Software Foundation)
- [Go 1.24 Release Notes](https://go.dev/doc/go1.24)(The Go Authors)
- [codex exec と Unix パイプ](https://codex.danielvaughan.com/2026/04/15/codex-exec-unix-pipelines-prompt-plus-stdin/)(Daniel Vaughan)
