---
title: "自宅NW活動誌: スロットリング監視を入れたら即日で電源劣化を検出した話 等"
date: 2026-07-20T17:00:00+09:00
categories: ["homelab"]
tags: ["raspberrypi", "monitoring", "hostapd", "wpa3"]
ai_assisted: true
draft: false
---

自作監視ツールにメトリクスを追加したら、その日のうちに電源劣化を検出してしまった。ついでにRaspberry Pi 3でWPA3が使えるか実験して散った。今日一日の活動記録。

## メトリクス追加: swap / throttled

自宅Raspberry Piクラスタを監視している自作ツール (Go製・SSH pull型・エージェントレス) に、メトリクスを2種追加した。

- **swap使用量**: `free -b` で取得。メモリ収集と同一コマンドなので実行を統合
- **スロットリング状態**: `vcgencmd get_throttled` の生hex値を保存し、表示側でビットデコード

throttledのビットフラグは以下の通り。

| bit | 意味 |
|-----|------|
| 0 | 低電圧 (現在) |
| 1 | ARM周波数制限 (現在) |
| 2 | スロットリング中 (現在) |
| 16 | 低電圧 (過去) |
| 17 | 周波数制限 (過去) |
| 18 | スロットリング (過去) |

アラートは**現在ビット (bit0-2) のみ対象**にした。過去ビットは再起動までクリアされないので、対象にすると鳴りっぱなしになる。この設計判断が後で効いてくる。

あわせてメモリ利用率の表示・アラートも追加。利用率はDBに保存せず、`(total - available) / total` を表示・判定時に都度計算する方式にした。`used / total` だとキャッシュ込みで常時高く出て、1GiBノードではアラートが無意味化するため、available基準が実態に合う。

## 監視が初日から仕事をした

デプロイ後ほどなくして、AP役ノードの1台 (Pi 3、以下AP#1) からthrottledアラートが鳴り始めた。

```
アラート種別: throttled 状態: 発生 値: 0x50005
アラート種別: throttled 状態: 復旧 値: 0x50000
```

`0x50005` = bit0 (低電圧・現在) + bit2 (スロットリング中) + 過去ビット。**稼働中に本当に低電圧が起きている**。

カーネルログで裏取りすると:

```
14:04:36 hwmon hwmon1: Undervoltage detected!
14:04:43 hwmon hwmon1: Voltage normalised
14:14:48 hwmon hwmon1: Undervoltage detected!
14:14:54 hwmon hwmon1: Voltage normalised
(以下、約10分間隔で継続、悪化時は1〜2分間隔)
```

各回4〜6秒で正常化するものの、慢性的に低電圧が発生していた。

### 原因: 4〜5年物のMicroUSB給電

このノードはMicroUSB給電のPi 3世代で、多ポートUSB充電ハブから4〜5年物のケーブルで給電していた。

- Pi 3B+の推奨は5V/2.5A。充電ハブは1ポートあたりの上限が別にあり、スマート充電のネゴシエーションがPiと相性の悪いケースもある
- MicroUSBケーブル自体が電圧降下の名所。経年劣化で状況はさらに悪化する
- WiFi APとして送信負荷が乗ると消費電力が跳ね、電圧が沈む

対処はケーブル交換から。選定条件は「2.4A以上対応明記 + 18〜20AWG + できるだけ短く」。ただし2026年現在、MicroUSBケーブルの選択肢はType-C移行でかなり痩せている。まず1本交換して24時間観察し、効果があれば全ノード分を展開する予定。

所詮ラズパイで構成されているネットワークなのでどこまで頑張るか、

天井は低いとは思うがまあ暇つぶしと現実を見ながらこれからも監視を手探りで追加、改修などを行いつつ、記事にしていきたい。

## Pi 3でWPA3は使えるか → 使えない (確定)

せっかくなので、以前から気になっていたWPA3にも挑戦した。hostapdは2.10でWPA3対応版。あとはドライバ (brcmfmac) 次第。

結果: **完敗**。

```
nl80211: kernel reports: key setting validation failed
wlan0: Could not connect to kernel driver
Interface initialization failed
```

`wpa_key_mgmt=WPA-PSK SAE` + `ieee80211w=1` を入れた瞬間、hostapdがwlan0を初期化できずAP-DISABLEDで死ぬ。SAEを諦めてPMF (`ieee80211w=1`) 単体でも同じエラー。**brcmfmacのAPモードは管理フレーム保護系の鍵設定を丸ごと受け付けない**、で一貫した挙動だった。

Pi 3 (内蔵WiFi) のAPとしての実効上限は以下となる。

```
wpa=2
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP
ap_isolate=1
```

WPA2-PSK + CCMP + クライアント間隔離。deauth攻撃への耐性 (PMF) は積めない。WPA2-PSKの実質的な弱点はハンドシェイクキャプチャからのオフライン辞書攻撃なので、長くランダムなパスフレーズで補うのが現実解。

### おまけ: hostapdがコケてBridgeが宙に浮いた結果

端末のIPとDNSMASQの情報があべこべになり、SSH不能に陥った。

hostapdが死ぬとwlan0がブリッジ (br0) に入らず、br0がDHCPを取り直して別IPを掴む。結果、旧IPと新IPが同居する半端な状態になり、pingは通るのにSSHが繋がらない・ICMPリダイレクトが飛び交う、という混乱が発生した。DNSが新IPを指していることに気づいて事なきを得た。

APは2台冗長構成なのでWiFi自体は無事だったが、突然SSHできなくなって混乱した。

## まとめ

- 自作監視へのthrottled追加は導入即日で電源劣化を検出。Pi運用ならthrottled監視は実質必須
- 電源系の異常は「ブート時だけ」と思い込まず、稼働中のログも時系列で確認する
- Pi 3 (brcmfmac) のAPモードはWPA3-SAEもPMFも不可。セキュリティ要件が上がったらハード更新が必要
- AP買い替え理由リスト: WPA3 / PMF / 5GHz / MicroUSB給電。候補はOpenWrt機か中古業務用AP。買う前にQEMUのOpenWrt x86イメージで砂場を作る手もある

監視ツールに機能を足したら、その機能が初日から障害を見つけ、調査の過程でハードの限界も一つ確定した。費用対効果の高い一日だった。
