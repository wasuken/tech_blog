---
title: "WireGuard 経由で UNEXT が見れない問題を解決した話"
date: 2026-06-06T00:00:00+09:00
draft: false
ai_assisted: true
tags: ["WireGuard", "iptables", "インフラ", "ネットワーク"]
categories: ["個人開発"]
description: "自宅ラズパイをルーターにして WireGuard VPN 経由で通信する環境で UNEXT が見れなかった。原因は VPS の IP が動画配信サービスにブロックされていたこと。iptables で UNEXT の IP 帯だけ WG をバイパスして解決した話。"
---

## 環境

- ラズパイ（Linux）がルーター兼 WireGuard クライアント
- 配下のデバイス（タブレット等）は eth0 経由でラズパイを通してインターネットへ
- ラズパイは wlan0 で ISP ルーター（192.168.x.1）に接続
- 全トラフィックを WireGuard（wg0）経由で VPS に流す構成
- VPS は国内 VPS サービス

```
タブレット(192.168.x.x)
  └─ eth0 ─ ラズパイ(ルーター)
               ├─ wlan0 ─ ISPルーター ─ インターネット
               └─ wg0 ─ VPS(国内) ─ インターネット
```

## 症状

- WireGuard 経由の WiFi で UNEXT が一切見れない
- YouTube・Amazon Prime Video は問題なし
- UNEXT の生配信は見れる、VOD だけ駄目

## 調査

### tcpdump で通信を確認

タブレットが UNEXT に接続しようとしたタイミングで tcpdump を仕掛けた。

```bash
sudo tcpdump -i eth0 -n 'src <タブレットIP> or dst <タブレットIP>' 2>/dev/null
```

UNEXT のサーバーへの SYN が**2回送られているが SYN-ACK が返ってこない**ことを確認。接続確立できていない。

### VPS から UNEXT に繋がるか確認

```bash
# VPS上で実行
curl -v --max-time 5 https://unext.jp
```

**タイムアウト。** VPS から UNEXT に繋がらない。

### ISP 直接（wlan0 経由）では繋がるか確認

```bash
# ラズパイ上で wlan0 を明示して実行
curl -v --interface wlan0 https://unext.jp --max-time 5
```

TLS ハンドシェイクまで到達。**繋がる。**

### VPS の IP を確認

```bash
curl http://ip-api.com/json/<VPS_IP>
```

結果は日本国内 IP だった。地域制限の問題ではない。

## 原因

国内 VPS の IP レンジが **UNEXT にブロックされていた**。

動画配信サービスはデータセンター・VPS・クラウドの IP レンジをまるごとブロックすることがある。地域制限というよりも、**ホスティング IP 自体を弾く**ポリシーによるもの。

生配信が見れていた理由は、生配信サーバーの IP 帯が別系統だったため。

・・・と思われる。

あくまでも予想。私のVPSのIPだけブロックされてるかもしれない。

そこはわからん。

## 解決策

UNEXT の IP 帯だけ WireGuard をバイパスして、ISP 直接（wlan0）で通信させる。

### iptables で UNEXT の IP 帯だけ wlan0 に MASQUERADE

```bash
sudo iptables -t nat -I POSTROUTING -s 192.168.x.0/24 -d 103.140.112.0/24 -o wlan0 -j MASQUERADE
sudo iptables -t nat -I POSTROUTING -s 192.168.x.0/24 -d 202.32.8.0/24 -o wlan0 -j MASQUERADE
```

### 永続化（wg0.conf に追記）

`/etc/wireguard/wg0.conf` の PostUp / PostDown に追記する。

```ini
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu; iptables -t nat -A POSTROUTING -s 192.168.x.0/24 -o %i -j MASQUERADE; iptables -t nat -A POSTROUTING -s 192.168.x.0/24 -d 103.140.112.0/24 -o wlan0 -j MASQUERADE; iptables -t nat -A POSTROUTING -s 192.168.x.0/24 -d 202.32.8.0/24 -o wlan0 -j MASQUERADE

PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -D FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu; iptables -t nat -D POSTROUTING -s 192.168.x.0/24 -o %i -j MASQUERADE; iptables -t nat -D POSTROUTING -s 192.168.x.0/24 -d 103.140.112.0/24 -o wlan0 -j MASQUERADE; iptables -t nat -D POSTROUTING -s 192.168.x.0/24 -d 202.32.8.0/24 -o wlan0 -j MASQUERADE
```

反映：

```bash
sudo wg-quick down wg0 && sudo wg-quick up wg0
```

## 結果

タブレットの UNEXT アプリで VOD が再生できるようになった。YouTube・Prime Video も引き続き問題なし。

## 余談：resolv.conf が壊れていた

調査中に `/etc/resolv.conf` の記述が不正であることも発覚。

```
# 壊れてた
nameserver 127.0.0.1,8.8.8.8

# 正しい（1行1エントリ）
nameserver 127.0.0.1
nameserver 8.8.8.8
```

`chattr +i` で上書き防止しておいた。

## まとめ

| 確認コマンド | 目的 |
|---|---|
| `tcpdump -i eth0 -n 'src/dst <IP>'` | どのIPと通信してるか |
| `curl --interface wlan0 <URL>` | WGバイパスして直接疎通確認 |
| `curl http://ip-api.com/json/<IP>` | IPの国・AS確認 |
| `ip route show table 51820` | WGのポリシールーティング確認 |
| `iptables -t nat -I POSTROUTING ... -o wlan0 -j MASQUERADE` | 特定IP帯だけWGバイパス |

特定サービスだけ VPN をバイパスしたい場合は、iptables の POSTROUTING で宛先 IP 帯を指定して wlan0 に MASQUERADE するのが手っ取り早い。UNEXT の IP 帯が変わったら tcpdump で新しい IP を確認して追加する必要がある。
