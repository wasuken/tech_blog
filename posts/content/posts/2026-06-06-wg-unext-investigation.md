---
title: "WireGuard 経由で UNEXT が見れない問題を調査した話【調査編】"
date: 2026-06-06T00:00:00+09:00
draft: false
tags: ["WireGuard", "iptables", "インフラ", "ネットワーク", "tcpdump"]
categories: ["個人開発"]
description: "WireGuard 経由で UNEXT が見れない問題の調査プロセスを記録した記事。tcpdump・iptables・ルーティングテーブルを使ってどう原因を絞り込んだかの思考トレース。"
---

解決編はこちら → [WireGuard 経由で UNEXT が見れない問題を解決した話 | 怠惰技術ブログ](https://techblog.wasutech.dev/posts/wg-unext-solution/)

## 概要

WireGuard 経由の WiFi で UNEXT の VOD だけ見れないという問題を調査した。症状・仮説・コマンド・結果の思考トレースを残しておく。

## 最初の症状整理

- UNEXT の VOD が見れない（くるくるのままタイムアウト）
- 生配信は見れる
- YouTube・Prime Video は問題なし

この時点での仮説：

1. MTU の問題（大きいパケットが通らない）
2. QUIC（UDP）の問題
3. DRM 認証の問題
4. VPS 側のブロック

## Step 1: MTU を疑う

WireGuard はオーバーヘッドがあるので MTU が小さくなる。大きいパケットが詰まってないか確認。

```bash
ping -M do -s 1400 8.8.8.8
```

結果：

```
From 192.168.x.254 icmp_seq=1 Frag needed and DF set (mtu = 1420)
ping: sendmsg: Message too long
```

**1400 バイトは通らない。** 1300 バイトは通った。MTU の壁が確認できた。

### TCPMSS clamping を確認

```bash
sudo iptables -t mangle -L FORWARD -n -v
```

```
TCPMSS  tcp  --  *  *  0.0.0.0/0  0.0.0.0/0  tcp flags:0x06/0x02 TCPMSS clamp to PMTU
```

すでに設定済みだった。TCP はケアされている。

## Step 2: QUIC（UDP）を疑う

TCPMSS clamping は TCP にしか効かない。UNEXT が QUIC（HTTP/3）を使っていたら効果がない。

```bash
sudo tcpdump -i any -n 'udp port 443' 2>/dev/null | head -20
```

結果：UDP 443 のトラフィックあり。QUIC を使っていることを確認。

### QUIC をブロックして TCP にフォールバックさせる

```bash
sudo iptables -I FORWARD -p udp --dport 443 -j DROP
```

YouTube が速くなった。Prime Video も見れた。しかし **UNEXT は依然タイムアウト**。

QUIC が原因ではないと判断。

## Step 3: 通信先 IP を特定する

Kindle（タブレット）の IP を特定してから、UNEXT 再生中のトラフィックを絞り込む。

```bash
sudo tcpdump -i eth0 -n 'src <タブレットIP> or dst <タブレットIP>' 2>/dev/null
```

再生ボタンを押した瞬間のログ：

```
<タブレットIP>.47328 > 192.168.x.254.53: A? scene01c.nxtv.jp.
<タブレットIP>.36148 > 103.140.112.106.443: Flags [S], seq ...
<タブレットIP>.36148 > 103.140.112.106.443: Flags [S], seq ...  ← SYN再送
```

`103.140.112.106` への **SYN が2回送られているが SYN-ACK が返ってこない**。接続確立できていない。

## Step 4: VPS から直接疎通確認

ラズパイ経由（WireGuard 経由）で繋がらないなら、VPS 自体が UNEXT に繋がるか確認する。

```bash
# VPS上で実行
curl -v --max-time 5 https://unext.jp
```

**タイムアウト。** VPS から UNEXT に到達できない。

## Step 5: ISP 直接（WireGuard バイパス）で確認

WireGuard を経由しない場合はどうか。

```bash
# ラズパイ上で wlan0 を明示
curl -v --interface wlan0 https://unext.jp --max-time 5
```

**TLS ハンドシェイクまで到達。繋がる。**

これで「WireGuard（VPS）経由だと UNEXT に繋がらない」ことが確定した。

## Step 6: VPS の IP を確認

国外 IP が原因の地域制限を疑って VPS の IP を調べた。

```bash
curl http://ip-api.com/json/<VPS_IP>
```

結果は**日本国内 IP**だった。地域制限ではない。

## Step 7: 原因の結論

国内 IP なのに UNEXT に繋がらない → **VPS・データセンターの IP レンジごとブロックされている**。?

動画配信サービスはクラウド・VPS の IP レンジをまるごとブロックするポリシーを持っていることがある。Netflix などと同じ仕組み。

生配信が見れていた理由は、配信サーバーの IP 帯が VOD サーバーと異なるため、ブロック対象外だったと考えられる。

あくまでも推測だが、調べようがない。

とはいえ、今回は良い暇つぶしになった。

## 調査で使ったコマンドまとめ

| コマンド | 何を確認したか |
|---|---|
| `ping -M do -s <size> <IP>` | MTU の壁がどこにあるか |
| `iptables -t mangle -L FORWARD -n -v` | TCPMSS clamping の有無 |
| `tcpdump -i any -n 'udp port 443'` | QUIC トラフィックの有無 |
| `iptables -I FORWARD -p udp --dport 443 -j DROP` | QUIC をブロックして TCP フォールバック確認 |
| `tcpdump -i eth0 -n 'src/dst <IP>'` | 特定デバイスの通信先確認 |
| `curl -v --max-time 5 https://unext.jp` | VPSから直接疎通確認 |
| `curl --interface wlan0 <URL>` | WGバイパスして直接疎通確認 |
| `curl http://ip-api.com/json/<IP>` | IPの国・AS確認 |

## 解決編

調査で原因が特定できたので、iptables で UNEXT の IP 帯だけ WireGuard をバイパスして解決した。

解決編はこちら → [WireGuard 経由で UNEXT が見れない問題を解決した話 | 怠惰技術ブログ](https://techblog.wasutech.dev/posts/wg-unext-solution/)
