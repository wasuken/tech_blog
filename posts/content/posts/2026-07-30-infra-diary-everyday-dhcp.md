---
title: "自宅NW活動誌：固定IP撤去とdnsmasq移行で名前解決を建て直した話"
date: 2026-07-30T09:10:00+09:00
draft: false
ai_assisted: true
tags: ["ネットワーク", "Linux", "dnsmasq", "インフラ", "DHCP"]
categories: ["技術"]
description: "電源断からの復旧をきっかけに、自宅セグメントのIP管理をdnsmasqのdhcp-hostに一元化した記録。"
---

## TL;DR

電源断からの復旧で名前解決がおかしくなり、調べていったら「dnsmasqがhosts変更を未反映」「gw-a/gw-bがローカル固定IPとdns01のDNS管理で二重管理になっていた」の2つが絡んでいたという話。ついでに「自ホスト名pingが`127.0.1.1`を返す」件は仕様通りで実害なしと確認した。全部片付いたので、今回は「詰んだ」話ではなく「ハマったけど直した」話。

## 環境

- セグメント: `home.local`（`192.168.20.0/24`）
- DNS/DHCP: dns01（dnsmasq）
- Keepalived構成のGW: gw-a, gw-b
- クライアント側ネットワーク管理: nmcli

## 発生した問題

電源断→起動後、あるノード（client01）から複数ノードにSSHできなくなった。調査を進める過程で、そもそもdns01自身の名前解決（`dns01`という名前そのもの）が引けなくなっていることが判明。`/etc/hosts`に手動でエントリを足しても、他ノードから引けない状態が続いた。

原因は1つではなく、「gw-a/gw-bがローカル固定IP設定＋hosts記載を持っていたこと」と「dns01側のhostsエントリ消失」が絡み合っていた。

## 対応1：dnsmasqがhosts変更を反映していなかった

**症状**: `/etc/hosts`にエントリを書いても、他ノードからdnsmasq経由で引けない。

**原因**: dnsmasqは`/etc/hosts`を起動時に読み込むだけで、実行中のファイル変更を自動では拾わない。dhcp-hostsfileやdhcp-optsfileなどのDHCP関連設定ファイルは変更時にSIGHUPシグナルを送ることでdnsmasqに再読み込みさせられる仕様になっており、`/etc/hosts`についても同様に、SIGHUPを受け取るとキャッシュをクリアした上で再読み込みする挙動になっている。つまり「編集しただけ」では反映されず、シグナルを送る操作が別途必要というのがハマりどころだった。

**対処**: `systemctl restart dnsmasq`（reload = SIGHUP相当）で即解決。

> **学び**: `/etc/hosts`をdnsmasq向けに編集したら、反射的に再起動する習慣をつける。

## 対応2：gw-a/gw-bを固定IP運用からdhcp-host予約運用に切り替え

**目的**: ローカル固定IP（dhcpcd.conf/nmcli static）とdns01のDNS管理が二重管理になっていたのを解消し、IP管理をdns01（dnsmasq）側に一元化する。

### 手順（実施順が重要）

**1. dns01側を先に設定**

`/etc/dnsmasq.conf`（またはdhcphostsファイル）に予約を追記する。

```
dhcp-host=<MACアドレス>,192.168.20.3   # gw-a
dhcp-host=<MACアドレス>,192.168.20.4   # gw-b
```

MACアドレスの大文字/小文字はdnsmasqでは区別されない（`AA:BB:...`でも`aa:bb:...`でも可）。区切り文字は`:`固定。設定後は`systemctl restart dnsmasq`。

**2. クライアント側（gw-a/gw-b）を動的取得に変更**

nmcli管理の場合は以下の通り。

```bash
nmcli connection show   # 対象接続名を確認
nmcli connection modify "<接続名>" ipv4.method auto
nmcli connection modify "<接続名>" ipv4.addresses ""
nmcli connection modify "<接続名>" ipv4.gateway ""
nmcli connection modify "<接続名>" ipv4.dns ""
nmcli connection up "<接続名>"
```

確実に反映させるため最終的に`reboot`。

**なぜこの順序なのか**: dns01側の予約が先に存在しないと、クライアントが動的取得に切り替わった瞬間にDHCPレンジ（20.20-20.99）から別のIPを掴むリスクがある。dns01→クライアントの順で作業することで、IPが変わらないことを保証できる。

## 対応3：自ホスト名pingが127.0.1.1を返す件（仕様、実害なし）

**症状**: `ping gw-a.home.local`は正しく`192.168.20.3`を返すが、`ping gw-a`（短縮名）は`127.0.1.1`を返す。

**原因**: Debian/Raspbian系はホスト名設定時に`/etc/hosts`へ自動で`127.0.1.1 <hostname>`のエントリが入る。`nsswitch.conf`の`hosts`エントリはデフォルトで`files dns`となっており、システムはまず`/etc/hosts`ファイルを、次にDNSサーバーを参照する仕様になっているため、短縮名での問い合わせはDNS（dns01）に届く前にこのローカルエントリで即答されてしまう。

FQDN（`*.home.local`）の場合は`/etc/hosts`に一致エントリがないため、正常にdns01へ問い合わせが飛び正しいIPが返る。自分自身から自分の短縮名を引く場合のみ発生する現象で、他ノードからの短縮名解決には影響しない（hostsのローカルエントリはそのノード内でしか効かないため）。

実害なし。気になる場合のみ`/etc/hosts`の`127.0.1.1`行を削除すれば解消するが、必須対応ではない。

## まとめ：対応早見表

| 問題 | 原因 | 対処 |
|---|---|---|
| dns01の名前解決不全 | dnsmasqがhosts変更を未反映 | `systemctl restart dnsmasq` |
| gw-a/gw-b IP管理の二重化 | ローカル固定IP＋dns01 DNS管理が併存 | dns01側dhcp-host予約 → クライアント側を動的取得に切替（この順序で実施） |
| 自分自身へのping結果が127.0.1.1 | `/etc/hosts`の`127.0.1.1 <hostname>`エントリがhosts優先で先に解決される | 仕様通りの挙動、実害なし |

## 所感

**「編集したら反映されるはず」という思い込みが一番のハマりどころだった。** dnsmasqに限らずデーモン系は「ファイルを書き換える」と「プロセスに反映させる」が別工程であることが多い。この手の「設定ファイルは正しいのに動きが変わらない」系は、まずシグナル/reload周りを疑う癖をつけておきたい。

IP管理の一元化については、二重管理を解消したことで今後の構成変更が「dns01だけ見ればいい」状態になったのは地味に楽になった。

## 参考

- [dnsmasqのSIGHUP挙動（Gentoo Wiki）](https://wiki.gentoo.org/wiki/Dnsmasq)
- [dnsmasqのhosts/DHCP関連ファイルreload仕様（Debian bugtracker）](https://groups.google.com/d/topic/linux.debian.bugs.dist/Q5yMgR5Aplc)
- [nsswitch.confのhosts解決順序（Debian公式リファレンス）](https://www.debian.org/doc/manuals/debian-reference/ch05.en.html)
