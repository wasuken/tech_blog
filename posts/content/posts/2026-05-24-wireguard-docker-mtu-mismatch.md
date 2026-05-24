---
title: "WireGuard + Docker で謎のタイムアウトが起きたら MTU を疑え"
date: 2026-05-24T00:00:00+09:00
draft: true
tags: ["WireGuard", "Docker", "インフラ", "ネットワーク", "さくらのクラウド"]
categories: ["個人開発"]
description: "WireGuard VPN 越しに Docker コンテナが外部 API を叩くと断続的にタイムアウトする。原因は MTU のミスマッチ。ホスト・WireGuard・Docker の3層をすべて揃えて解決した話。"
---

個人開発のツールをさくらのクラウドの VPS に乗せて、WireGuard で自宅からアクセスできるようにしていた。  
ある日、Docker コンテナ内から外部 API（EDINET、Gemini）を叩くと**断続的にタイムアウトする**という謎の現象が起きた。

ローカルから curl すると普通に返ってくる。コンテナ内から小さいリクエストはOK。大きいレスポンスになると死ぬ。典型的な MTU 問題だった。

## 何が起きていたか

ネットワークのパスはこうなっている：

```
コンテナ → Docker bridge → WireGuard → インターネット
```

それぞれが持つ MTU：

| レイヤー | デフォルト MTU |
|---|---|
| 通常の Ethernet | 1500 |
| WireGuard | 1420 前後（オーバーヘッド分削る） |
| Docker bridge | **1500**（WireGuard を無視） |

Docker はホストの物理 NIC の MTU を見て bridge を作るが、**WireGuard インターフェース経由でルーティングされることを知らない**。  
結果、1500 バイトのパケットを送り出すが WireGuard で分断が起き、フラグメントされたパケットが行方不明になる。

## 解決策：3層すべての MTU を揃える

### 1. WireGuard クライアント設定

```ini
[Interface]
MTU = 1280
```

### 2. WireGuard サーバー設定

```ini
[Interface]
MTU = 1280
```

### 3. Docker の MTU を固定する

`/etc/docker/daemon.json` を作成（または編集）：

```json
{
  "mtu": 1280
}
```

```bash
sudo systemctl restart docker
```

### 4. compose.yml の network にも念のため指定

```yaml
networks:
  default:
    driver: bridge
    driver_opts:
      com.docker.network.driver.mtu: "1280"
```

## なぜ 1280 か

WireGuard のオーバーヘッドは約 60 バイト（IPv4）。  
1500 - 60 = 1440 でも動くことはあるが、IPv6 を考慮すると 1280 が安全圏。  
また `daemon.json` で設定した値は既存の network には反映されないので、`docker network prune` してコンテナを再作成する必要がある。

## まとめ

- WireGuard 越しに Docker を動かすときは MTU ミスマッチに注意
- **ホスト（WireGuard）・`daemon.json`・`compose.yml`** の3箇所を統一する
- 症状が「大きいレスポンスだけタイムアウト」なら MTU を真っ先に疑う
