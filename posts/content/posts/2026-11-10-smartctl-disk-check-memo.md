---
title: "smartctlでディスクの健康状態と使用年数をざっくり確認するメモ"
date: 2026-11-10T20:00:00+09:00
draft: false
ai_assisted: true
tags: ["Linux", "インフラ", "ストレージ", "smartctl"]
categories: ["技術"]
description: "smartctlで健康状態・SMART属性・自己診断を見る流れと、通電時間から使用年数を出す方法、SSD/HDDの判別と長期利用の比較をまとめた雑メモ"
---

古めのディスクの状態を見る機会があったので、確認の流れをメモしておく。雑メモなので、網羅性より「次に見るときすぐ使えること」を優先している。

## TL;DR

- 流れは **全体の健康状態（`-H`）→ 詳細属性（`-A`）→ 自己診断テスト（`-t` / `-l selftest`）**
- `PASSED` でも安心しきらない。属性 5 / 197 / 198 が **0から動いていないか** を見る
- 使用年数は **Power_On_Hours**（通電時間）から出せる
- SSDとHDDの優劣は **使い方で逆転する**。常時稼働ならSSD、通電しない長期保管ならHDD。どちらでもバックアップは必須

## 対象デバイスの確認

まず対象のデバイス名を確認する（`/dev/sda`、NVMeなら `/dev/nvme0n1` など）。

```bash
# ディスク一覧（ROTA列はあとで使う）
lsblk -d -o name,rota
```

## 1. 全体の健康状態

```bash
sudo smartctl -H /dev/sda
```

| 結果 | 意味 | 対応 |
|---|---|---|
| **PASSED** | 現時点で大きな問題なし | ただし詳細属性は別途確認する |
| **FAILED** | 故障状態、または故障が差し迫っている | 直ちにバックアップして交換を検討 |

`PASSED` は「閾値を割っていない」程度の意味合いなので、**これだけで健康と判断しない**。

## 2. 詳細SMART属性

```bash
sudo smartctl -A /dev/sda
```

HDD（ATA/SATA）で特に見る属性は次の4つ。名称やRAW値の解釈にはベンダー・機種による差があり、NVMe SSDでは別の項目（後述）を見る。

| ID | 属性名 | 見方 |
|---|---|---|
| 5 | Reallocated_Sector_Ct | 代替処理済みセクタ数。増加は不良セクタの存在を示す |
| 197 | Current_Pending_Sector | 代替処理保留中のセクタ数。0以外は警告サイン |
| 198 | Offline_Uncorrectable | オフライン修正不能セクタ数。0以外は深刻 |
| 199 | UDMA_CRC_Error_Count | CRCエラー数。ディスク本体ではなく **SATAケーブルの接触不良・故障** を示すことが多い |

199だけ性質が違う点が地味に重要で、SATAの通信経路（ケーブル、コネクタ、コントローラなど）の問題を疑う属性。ここが増えているときはディスクを疑う前にケーブルを替えるのが先、とよく言われる。累積値なので、過去の分なのか、今も増えているのかを見る。

## 3. 自己診断テスト

```bash
# 短時間テスト（通常は数分以内。機種による）
sudo smartctl -t short /dev/sda

# 長時間テスト（容量により数時間）
sudo smartctl -t long /dev/sda

# 結果確認
sudo smartctl -l selftest /dev/sda
```

テストはバックグラウンドで走る。所要時間の目安は `smartctl -c /dev/sda` で確認できる（[ArchWiki](https://wiki.archlinux.org/title/S.M.A.R.T.)）。

結果は **最新の1行目** を見る。

| 表示 | 意味 |
|---|---|
| Completed without error | 正常完了、エラーなし |
| Completed: read failure | 読み取りエラー。不良セクタの可能性 |
| Interrupted | 中断（サスペンドや手動中止など） |
| Self-test in progress | 実行中（Remainingで残り%が出る） |

`LBA_of_first_error` が空白または `-` ならエラーなし。数値が入っていれば、その位置で読み取りエラーが出ている。

### 特殊ケース

- **NVMe SSD**：`sudo smartctl -a /dev/nvme0` で見る（種別は自動判別されるが、`-d nvme` を付けてもよい）。SATAの属性5・197・198ではなく、Critical Warning / Available Spare / Percentage Used / Media and Data Integrity Errors あたりを見る
- **ハードウェアRAIDカード配下**：直接見えないことがある。`storcli` / `MegaCli` か、`-d megaraid,N` などのパススルーを試す（手元で試していないので、書き方は環境に合わせて要確認）。対応状況はコントローラ依存（[smartmontools FAQ](https://www.smartmontools.org/wiki/FAQ)）

## 使用年数の確認

SMART属性の **Power_On_Hours（通電時間）** で見る。

```bash
sudo smartctl -A /dev/sda | grep Power_On_Hours
```

多くのHDDでは最終列の `RAW_VALUE` が通電時間（時間単位）になる。ただしRAW値の解釈はメーカー・機種で違うことがあるので、桁がおかしいときは単位を疑う。

製造日はディスク表面のラベルを見るのが最も確実で、ソフトからは読み取りにくい。

### 年数換算の例

手元のHDDで Power_On_Hours が **37,647時間**、`-H` は PASSED、自己診断もエラーなしだった。これを年数に直すとこうなる。

| 稼働パターン | 換算 |
|---|---|
| 24時間連続稼働 | 約 **4.3年** |
| 1日8時間稼働 | 約 **12.9年** |

HDDの故障率は使用年数、機種、運用環境で変わるので、「何年で壊れる」と一律には言えない。ただし37,647時間は相応に長い。

PASSEDでエラーなしでも、**このくらい回っていたら「そろそろ代替わりを考え始める時期」** と見ておく。やることは地味で、

- バックアップを最新にする
- 属性5・197を定期的に見る
- 交換用のディスクを用意しておく

属性5・197が **0から増え始めたら故障の前兆** と見てよい。

## SSDかHDDか

```bash
lsblk -d -o name,rota
```

`ROTA` 列が **0ならSSD（非回転）、1ならHDD（回転）**。

ただし仮想マシン、一部のUSBメモリ、特殊なコントローラでは誤表示することがある。確実にしたいなら smartctl で見る。

```bash
sudo smartctl -a /dev/sda | grep -i "Rotation Rate"
```

`Solid State Device` ならSSD、`7200 rpm` などならHDD。

## SSDとHDD、長く使うならどっち？

単純にどっちが上とは言えず、**使い方で結論が変わる**。

| 観点 | 有利 | 理由 |
|---|---|---|
| 常時稼働・日常使用 | **SSD寄り** | 速度、静音性、衝撃への強さで有利。故障率の差は条件次第 |
| 通電しない長期保管 | **SSDは注意** | SSDは電荷が抜けてデータが消えるリスクがある。HDDも定期的な読み出し確認は必要 |

### 常時稼働の場合

メーカーの公表値やデータセンターの統計を見ると、おおむねSSDのほうが故障率は低め、という傾向が出る。ただし、ドライブの年齢を揃えて比べると差はかなり小さくなる、という報告もある（[Backblaze](https://www.backblaze.com/blog/are-ssds-really-more-reliable-than-hard-drives/)）。故障率だけでSSDを選ぶのは微妙で、速度や静音性込みで選ぶのが現実的。

壊れ方については、HDDはヘッドやモーターなどの物理的な故障があり、SSDはNANDの書き換え寿命に加えてコントローラや電源回路の故障もある。**どちらも予兆なく突然使えなくなる可能性がある** ので、壊れ方でバックアップの方針を変えるより、どちらでも前提にして取っておくほうがいい。

### 長期保管の場合

SSDは通電しないと電荷が抜けていき、放置期間が長いとデータが消えるリスクがある。保持期間は個体、温度、書き込み量で大きく変わる。JEDECの規格には電源断時のデータ保持の要件があるが、あくまで決められた条件での要件で、実際の保持期間はNANDの摩耗状態や温度で変わる。

HDDは磁気記録なので、通電なしでもデータが保持されやすい（経験則）。ただしHDDも、長期保管で機械部品が固着したり、読み出して初めてエラーに気づいたりすることがある。

### 結論

- PCの起動ドライブ → **SSD**
- 大容量のバックアップ置き場 → コスト面でHDDも有力
- 長期間通電しない保管 → SSDはデータ保持に注意、HDDも定期的な検証が必要
- 媒体の種類より大事なのは、**複数コピー、定期的な読み出し確認、必要に応じた媒体の更新**

## まとめ

| やりたいこと | コマンド |
|---|---|
| 健康状態 | `smartctl -H` |
| 属性（5/197/198/199、Power_On_Hours） | `smartctl -A` |
| 自己診断 | `smartctl -t short`（または `long`）→ `smartctl -l selftest` |
| SSDかHDDか | `lsblk -d -o name,rota` / `smartctl -a \| grep -i "Rotation Rate"` |

**SMARTは「壊れそう」の予兆を拾う道具であって、ディスクの寿命を保証もバックアップの代わりもしない。**

## 参考

- smartmontools project, "Short Selftest" — https://www.smartmontools.org/wiki/selftest_short
- smartmontools project, "FAQ" — https://www.smartmontools.org/wiki/FAQ
- ArchWiki, "S.M.A.R.T." — https://wiki.archlinux.org/title/S.M.A.R.T.
- Backblaze, "Are SSDs Really More Reliable Than Hard Drives?" — https://www.backblaze.com/blog/are-ssds-really-more-reliable-than-hard-drives/
