---
title: "AdGuard HomeをProxmox LXCに立ててTailscale経由でDNSブロックする"
date: 2026-04-06T07:00:00+09:00
draft: false
tags: ["proxmox", "adguard", "tailscale", "homelab", "dns"]
---

## 概要

はてな匿名ダイアリーとウーバーイーツをインフラレベルで封鎖したかった。
AdGuard HomeをProxmox LXCに立てて、Tailscale経由でDNSブロックする構成を作った。

## 環境

- Proxmox VE
- Tailscale導入済み
- AdGuard Home v0.108.0

## 手順

### 1. AdGuard Home LXCをスクリプト一発で作成

Proxmoxのノードシェルで以下を実行する。
```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/adguard.sh)"
```

[community-scripts/ProxmoxVE](https://github.com/community-scripts/ProxmoxVE)が提供するスクリプト。
LXCのコンテナ作成からAdGuard Homeのインストールまで全自動でやってくれる。

デフォルト構成はDebian 13、CPU 1コア、RAM 512MB、HDD 2GB。DNS用途なら十分。

### 2. LXCにTUNデバイスを追加する

TailscaleはWireGuardベースのVPNで、動作に`/dev/net/tun`が必要。

`/dev/net/tun`はLinuxの仮想ネットワークデバイス（TUNデバイス）。通常のネットワークデバイス（eth0等）は物理NICに紐づいているが、TUNはソフトウェアで作った仮想NIC。

Tailscaleは以下の流れで通信を処理する。

1. 通信をTailscaleプロセスが横取り
2. WireGuardで暗号化
3. 暗号化したパケットを相手に送る

この「横取り」の実装に`/dev/net/tun`を使う。TUNデバイスを通してカーネルのネットワークスタックとTailscaleプロセスがやり取りする仕組みになっている。

unprivileged LXCはセキュリティ上の理由でホストのデバイスに触れないようになっているため、明示的に`/dev/net/tun`をコンテナに見せてあげる必要がある。

Proxmoxのノードシェルで以下を実行してTUNを有効化する。

```bash
pct stop 106
echo "lxc.cgroup2.devices.allow: c 10:200 rwm" >> /etc/pve/lxc/106.conf
echo "lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file" >> /etc/pve/lxc/106.conf
pct start 106
```

`106`の部分は自分のLXCのIDに置き換える。IDは`pct list`で確認できる。

### 3. LXCにTailscaleを入れる

LXCのシェルに入って以下を実行する。
```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
```

認証URLが表示されるのでブラウザで開いてログインする。
認証後、TailscaleのMachines画面からAdGuard HomeのTailscale IPを確認しておく。

### 4. TailscaleのDNS設定にAdGuard HomeのIPを追加する

[Tailscale管理画面のDNS設定](https://login.tailscale.com/admin/dns)を開く。

- **Nameservers → Add nameserver → Custom** でAdGuard HomeのTailscale IPを追加
- **Override DNS servers** をONにする（これをONにしないとAdGuardが優先されない）
- デフォルトで入っているGoogle Public DNS（8.8.8.8等）は削除する

### 5. AdGuard Homeの管理画面でブロックルールを追加する

`http://<AdGuardのTailscale IP>:3000` にアクセスして管理画面を開く。

**Filters → Custom filtering rules** に以下の形式でルールを追加する。

```
||anond.hatelabo.jp^
||ubereats.com^
```

**Apply**を押せば即時反映される。

### 6. フィルタの自動更新

**Filters → DNS blocklists** の更新頻度はデフォルト24時間。そのままでOK。
AdGuard Home本体のアップデートはUIから手動で行う。

## まとめ

Tailscaleネットワーク内からDNSレベルで特定ドメインをブロックできるようになった。
ルーターの設定を触らずに済むのでホームルーター環境でも安心して導入できる。
