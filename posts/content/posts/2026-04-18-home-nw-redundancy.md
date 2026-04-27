---
title: "自宅NWをKeepalivedとdnsmasqで作り直した話"
date: 2026-04-18T19:00:00+09:00
draft: false
tags: ["homelab", "keepalived", "dnsmasq", "tailscale", "raspberry-pi", "network"]
categories: ["インフラ"]
---

## 背景

自宅のネットワーク構成が辛かった。

具体的には以下の問題があった。

- ルータが1台構成で、死んだら自宅NW全滅
- メンテナンスで定期的に落としたいとかあっても、コストが大きい
- DNS/DHCPがルータに同居していて、役割が混在している
- WireGuardで外部接続していたが、外部からの入り方がだるい。

特に「冗長化したい」という気持ちは何年も前からあったが、DNSとDHCPの同期という難問の前に何度も挫折してきた。

結論からいえば、DNS,DHCPの冗長化は諦めた。

諦めた上で出口のルータとDNS、DHCPは役割を分けた上で出口のルータとWIFIルータのみ冗長化を実施した。

## 設計方針

過去の失敗から学んだ一番の教訓はやはり「DNS/DHCPの冗長化は難しい」ということだ。

理論上可能ではあるが、私は万年初級自宅インフラエンジニアでいつまで経ってもできる目処が立たない。

それでいて、生成AIという過ぎた兵器を持ってしても、この問題は解決しなかったので、難しいというよりは不可能とした。

2台のサーバでDHCPを冗長化しようとすると、リースの同期問題が必ず発生する。同期ツールを使ったりしたが、どうしても同期しきれなかった。

そこで今回は思い切って諦めた上で役割を分離する方針にした。

```
ルータ（冗長化） → GWとTailscaleだけ担当
AP             → hostapdのみ
DNS/DHCP       → 専用機1台に集約（冗長化しない）
```

DNSとDHCPは単一障害点になるが、自宅ラボであれば許容範囲だと判断した。

**てかどうしようもない。**

## 構成

使用機材はすべてRaspberry Pi。封印していたタワーからも引っ張り出した。

| ホスト | 役割 |
|--------|------|
| ルータA (MASTER) | Keepalived + Tailscale |
| ルータB (BACKUP) | Keepalived + Tailscale |
| DNS/DHCP機 | dnsmasq専用 |
| AP x2 | WIFI冗長化 |

ネットワークセグメントはひとつ。VIPをデフォルトゲートウェイ兼DNSとして各クライアントに配布する。

## Keepalivedの設定

まずルータ2台にKeepalivedを入れる。

```bash
sudo apt install -y keepalived
```

MASTER側の設定:

```
vrrp_instance VI_1 {
    state MASTER
    interface lan-if
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass xxxxxxxx
    }
    virtual_ipaddress {
        192.168.xx.1/24
    }
}
```

BACKUP側は `state BACKUP` と `priority 90` に変えるだけ。シンプルだ。

起動後、MASTERがVIPを持ち、BACKUPが待機状態になることを確認した。

## iptablesの設定

ルータとしての役割を果たすにはMASQUERADEとDNS転送が必要だ。

```bash
# 外部への通信 (uplink-if経由で上流ルータへ)
sudo iptables -t nat -A POSTROUTING -o uplink-if -j MASQUERADE

# VIPへのDNSクエリをdnsmasq専用機へ転送
sudo iptables -t nat -A PREROUTING -d 192.168.xx.1 -p udp --dport 53 -j DNAT --to-destination 192.168.xx.5:53
sudo iptables -t nat -A PREROUTING -d 192.168.xx.1 -p tcp --dport 53 -j DNAT --to-destination 192.168.xx.5:53

# 永続化
sudo apt install -y iptables-persistent
sudo netfilter-persistent save
```

VIPに来たDNSクエリをdnsmasq専用機（192.168.xx.5）に転送する設計にした。こうすることで、クライアント側のDNS設定はVIPのまま変えなくて済む。

## dnsmasqの設定

DNS/DHCP専用機にdnsmasqを入れる。

```bash
sudo apt install -y dnsmasq
```

設定ファイル `/etc/dnsmasq.conf`:

```
port=53
interface=lan-if
domain-needed
domain=home.local
expand-hosts

server=8.8.8.8
server=8.8.4.4

dhcp-authoritative
dhcp-range=192.168.xx.20,192.168.xx.99,12h
dhcp-option=option:router,192.168.xx.1
dhcp-option=option:dns-server,192.168.xx.1
dhcp-option=option:domain-search,home.local
dhcp-leasefile=/var/lib/misc/dnsmasq.leases
```

### ハマりポイント1: bind-interfaces

最初 `bind-interfaces` を入れていたら、dnsmasqが `127.0.0.1:53` だけで待ち受けてしまい外部から到達できなかった。コメントアウトすることで全インターフェースで待ち受けるようになった。

### ハマりポイント2: avahi-daemon

`.local` ドメインはmDNS予約済みのため、avahi-daemonが干渉して名前解決ができなかった。全ノードでavahi-daemonを無効化して解決した。

```bash
sudo systemctl stop avahi-daemon
sudo systemctl disable avahi-daemon
```

### ハマりポイント3: expand-hosts

`/etc/hosts` に `192.168.xx.5 myhost` と書いても `myhost.home.local` で解決できなかった。`expand-hosts` ディレクティブを追加することで、hostsのエントリにドメインが自動付与されるようになった。

## TailscaleのSplit DNS設定

全ノードにTailscaleが入っているため、Tailscaleがresolv.confを上書きしてしまう問題があった。

Tailscaleの管理画面でSplit DNSを設定することで解決した。

- DNS → Nameservers → Add nameserver → Custom
- IPアドレス: 192.168.xx.5
- Restrict to domain: `home.local`

これにより `home.local` のクエリだけdnsmasq専用機に向けられ、それ以外はTailscale DNSが処理する。

## WIFIの冗長化

APは2台用意し、hostapdでそれぞれをアクセスポイントとして動かす。

```bash
sudo apt install -y hostapd
```

設定ファイル `/etc/hostapd/hostapd.conf`:

```
interface=wlan0
bridge=br0
driver=nl80211
ssid=home-wifi
hw_mode=g
wmm_enabled=1
auth_algs=1
wpa=2
wpa_passphrase=xxxxxxxx
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
ieee80211n=1
max_num_sta=15
beacon_int=200
```

2台はチャンネルだけ変える（例: ch6とch7）。同一チャンネルだと干渉するためだ。SSIDとパスワードは同一にする。

Bridgeモード（br0）で動かすことで、WIFI経由で繋いできたクライアントが有線側の192.168.xx.0/24セグメントにそのまま収容される。DHCPもdnsmasq専用機から取得できる。

```bash
# br0を作成してeth0をスレーブに
sudo nmcli con add type bridge ifname br0
sudo nmcli con add type bridge-slave ifname eth0 master br0

# hostapd有効化
sudo systemctl enable --now hostapd
```

クライアントは自動的に電波の強い方のAPに繋がる。DHCPの同期問題もなく、これが一番シンプルな冗長化だった。

ちなみにここらへんは全部前に試したときの設定残ってたのでちょちょいとなおしたら普通に動くようになった。

## 結果

フェイルオーバーテストとしてMASTERの電源を抜いたところ、数秒でBACKUPがVIPを引き継ぎ通信が復旧した。DNSもDHCPもVIP経由で継続して機能した。

構成のポイントをまとめると:

- **ルータの冗長化**: Keepalivedでシンプルに実現
- **DNS/DHCPは分離・単一化**: 同期問題を回避
- **DNATでVIPへの透過性を確保**: クライアント設定変更不要
- **avahi無効化とexpand-hosts**: `.local`ドメイン運用の必須設定
- **WIFI冗長化**: 同一SSID・チャンネル分け・Bridgeモードでシンプルに実現

## 今後の課題

- あくまでも今のNWとは別のSwitchなどを介して構築しているため、使っていない開発PCなどをこちらに移行するか考え中。
- 外形監視の仕組みを作りたい
- VPSにHeadscaleを構築してTailscaleをセルフホスト化したい

上２つは多分やる。最後のは多分やらないと思う。やる気がマックスファイアーならやる。
