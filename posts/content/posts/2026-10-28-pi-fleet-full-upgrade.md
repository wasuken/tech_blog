---
title: "自宅Raspberry Pi 8台を一斉に更新した話：資料より実機を見ろ"
date: 2026-10-28T09:00:00+09:00
draft: false
ai_assisted: true
tags: ["Raspberry Pi", "Linux", "Debian", "apt", "ネットワーク", "インフラ", "個人開発"]
categories: ["技術"]
description: "自宅NWのRaspberry Pi 8台をapt-cacher-ng経由でfull-upgradeした記録。資料と実機の食い違い、bullseye→bookwormの落とし穴、未解決のDNS問題まで。"
---

自宅ネットワークのRaspberry Piを全部まとめて更新した。

## TL;DR

- Pi 8台を `full-upgrade` して、カーネルを揃えた。
- 台数分のダウンロードが重かったので、`apt-cacher-ng` を1台立てた。上流から落としたのは1台目の約268MBが中心で、2台目以降は0〜1MBだった。
- 一番手こずったのは、資料に全部bookwormと書いてあったのに、実機の1台だけbullseyeだったこと。ここから連鎖的にハマった。

## 構成

8台とも自宅NWで動いてる。

| 役割 | 台数 | 備考 |
|---|---|---|
| ゲートウェイ | 2 | Keepalived（MASTER/BACKUP）、Tailscale GW |
| DNS/DHCP | 1 | dnsmasq |
| WiFi AP | 2 | hostapd、ブリッジ構成 |
| 監視ホスト | 1 | Pi 5 |
| 予備 | 2 | Pi3, Pi4 |

`apt update` は、各ノードが週1のsystemd timerで勝手に実行する設定にしてある。`upgrade` のほうは、月次で手動にしたかった。

## 資料は全部bookworm、実機は違った

更新の前に、まず各ノードのOSを確認した。自分で作った資料には全部bookwormと書いてあったので一応調査。

実機を見たら、DNS用の1台だけがbullseye（32bit）だった。

資料には、OSの版だけでなく、IP、WiFiのチャンネル、ドメインまで食い違いがあったので都度修正した(Claudeが)

ちまちま修正したりファイル化もいいけど少し前からclaudeのコネクタ機能を使ってNotionにガリガリ書いたり読んだりしてる。かなり便利

なおbullseyeは、Debian公式の告知では2026年8月31日にLTSも含めてサポートが終了している。

## DNS用ノードをin-placeでbullseyeからbookwormへ

前節に書いたとおり、このノードのbullseyeはすでにサポートが切れていた。

bullseyeからbookwormへのin-placeアップグレードは公式は勧めていない。
Raspberry Piの公式ニュース記事にはメジャーバージョンの更新はSDカードを焼き直すのが基本と書いてある。

それでも今回は、再インストールではなくin-placeで上げた。
できないわけではないし自宅NWには迂回路があって失敗しても致命傷にはならないと判断した。

落とし穴は4つあった。

### 1. `/boot` が `/boot/firmware` に変わる

bookwormのRaspberry Pi OSでは起動パーティションのマウント先が `/boot` から `/boot/firmware` に変わった。そのため、fstabを直す必要があった。

Raspberry Pi公式のconfig.txtのドキュメントに、Bookwormより前は起動パーティションは `/boot/` にあったという趣旨の注記がある（出典参照）。bullseyeからのアップグレード手順をまとめた個人のgistにも、同じ内容の手順が載っていた。実機でも、fstabを直さないと起動パーティションの扱いがおかしくなった。

### 2. 旧カーネルパッケージが止まっている

旧体系の `raspberrypi-kernel` は、更新が止まっていた。
新しい体系（`raspi-firmware` と `linux-image-rpi-v7`）を、明示的にinstallして切り替えた。

こういう自動で拾ってくれないパッケージ名の切り替えは、`full-upgrade` を流すだけでは終わらない。

### 3. sourcesのホスト名が旧ドメインのまま

`sources.list` に書いてあったホスト名が、旧ドメイン（`.org`）のままだった。更新が通らないので気づけた。

### 4. 固定IPのプロファイルがメモリ上にしかない

これは私のNW固有の問題で一番ひどかったのがこれ。

固定IPのNetworkManagerプロファイルが、`/run`（メモリ上）にしかなかった。起動のたびに自動作成のDHCPプロファイルが45秒間失敗する。その間、**DNS/DHCPが止まる**。

家のDNSとDHCPを担当しているノードなので再起動のたびに家のネットワークが45秒死ぬ。

`nmcli con modify` で `/etc` 側に保存し直して解消した。

| 症状 | 原因 | 対処 |
|---|---|---|
| 起動のたびにDNS/DHCPが45秒止まる | 固定IPのプロファイルが `/run` にしかない | `nmcli con modify` で `/etc` に保存 |
| 更新が通らない | sourcesが旧ドメイン | ホスト名を直す |
| 起動パーティションの扱いがおかしい | `/boot` → `/boot/firmware` | fstabを修正 |
| カーネルが上がらない | 旧パッケージが止まっている | 新体系を明示的にinstall |

## apt-cacher-ngを監視ホストに入れた

8台分のダウンロードは重い。そこで、監視ホスト（Pi 5）に `apt-cacher-ng` を入れた。
HTTPのリポジトリへのアクセスをキャッシュしてくれるプロキシだ。

設計で気をつけたのは**キャッシュが落ちてもaptが止まらないこと**。

クライアントは `Proxy-Auto-Detect` を使い、プロキシに届かなければ `DIRECT`（直接取得）に落とす。中身はだいたいこんなイメージ。

※記事用に簡略化していて実物とは少し違う。`nc` が無い環境では、先にインストールが必要。

```sh
#!/bin/sh
# aptから呼ばれる。標準出力にプロキシURL、または DIRECT を返す
# キャッシュ(3142番)に届けばそれ経由、届かなければ直接取得
if nc -z -w 1 192.168.X.Y 3142; then
  echo "http://192.168.X.Y:3142"
else
  echo "DIRECT"
fi
```

```
// /etc/apt/apt.conf.d/ 配下の設定イメージ
Acquire::http::Proxy-Auto-Detect "/usr/local/bin/detect-apt-proxy";
```

決めたことがいくつかある。

1. httpsのリポジトリ（TailscaleとDocker）は、キャッシュを通さず直接取得する。
2. プロキシはIP指定にした。短いホスト名が引けないノードがあったため。
3. arm64のノード（2台）は、armhfとはキャッシュを共有しない。アーキテクチャごとにパッケージが別。そりゃそうだ。

### httpsのリポジトリを通さなかった理由

apt-cacher-ngは、そのままではhttpsのリポジトリを扱えない。Debian wikiによると、`PassThroughPattern` でCONNECTトンネルを許可すれば通せるが、その場合も中身はキャッシュされない。許可しないと `403 CONNECT denied` で失敗する、という報告もある（Qubesのフォーラム）。

作者の公式ユーザーマニュアルにも、HTTPS（SSL/TLS）のリモートへのアクセスの節がある。CONNECTには対応しているが、ローカルの設定で有効にする必要がある。許可は `PassThroughPattern` に正規表現を書いて行う（例として挙がっている `.*` は、どのホストもどのポートも許可する）。HTTPSは接続の仕組み上、HTTPと同じようには中継キャッシュできない、という説明だった。

同じ節には、HTTPのURLに置き換えてキャッシュする方法（URLに `HTTPS///` を挟む、バックエンドの一覧に書く）も載っていたが、自分は試していない。自分は最初から通さない設計にしたので、トンネルも実際には試していない。なお、読めたのは古い版（0.9.1）のPDFで、最新版のマニュアルは取得できていない。

## `full-upgrade` で全ノードを更新

### `upgrade` では足りなかった

自分の環境では`upgrade` 系のコマンドではカーネル本体が上がらず`full-upgrade` を使ったら上がった。

manを読んだら`apt-get` と `apt` で `upgrade` の定義が違っていた。

| コマンド | `upgrade` の説明（manの趣旨） |
|---|---|
| `apt-get upgrade` | 入っているパッケージだけを上げる。まだ入っていないパッケージを新規に取得・インストールはしない（`--with-new-pkgs` を付ければ新規も許可される） |
| `apt upgrade` | 依存を満たすために必要なら、新規パッケージも入れる。ただし既存パッケージは削除しない。削除が必要になる更新は見送る |
| `full-upgrade` | `upgrade` と同じことをしつつ、必要なら既存パッケージを削除する |

自分が流したのは `apt upgrade` のほうでカーネル本体は、版名の入った新しい名前のパッケージとして出てくる。

最初は新規パッケージだから `upgrade` では入らないと理解していた。

でも、`apt upgrade` のmanには依存のために必要な新規パッケージは入れると書いてあるのでこの説明は `apt` のほうでは成り立たない。

かといって、事前の `-s` では、削除（`Remv`）は出なかったので削除が必要だったから見送られたということでもなさそう。

つまり、なぜ `apt upgrade` では上がらず `full-upgrade` で上がったのか私にはわからない。
kept backの一覧も記録していなかった。確実に言えるのは、`full-upgrade` で上がったことと、その前に `-s` で削除が無いことを確認したこと、の2点だけ。

### 事前に `-s` で確認した

`full-upgrade` は、必要なら既存パッケージを削除する。GWやAPの肝になるパッケージが巻き込まれたら困る。そのため、事前にシミュレーションで確認した。

```sh
# -s はシミュレーション。Remv(削除)の行が無いか、肝のパッケージが巻き込まれないかを見る
apt-get -s full-upgrade | grep -E '^Remv'
```

`Remv` が無いことを確認してから、本番を流した。

### 順番

止めても被害が小さいところから順に上げた。DNS用ノードは、前節でin-placeを先に済ませてあるので、この表には入っていない。

| 順 | 対象 | 理由 |
|---|---|---|
| 1 | 予備機 | 止まっても影響なし。手順の予行演習になる |
| 2 | WiFi AP（1台ずつ） | 同時に両方止めるとWiFiが全滅する |
| 3 | GWのBACKUP | 先にBACKUPで試す |
| 4 | GWのMASTER | VIPが切り替わる前提で |
| 5 | 監視ホスト（最後） | キャッシュ元なので、最後に回す |

各ノードで再起動後にカーネル版、サービス、NATルール、VIPを確認した。

監視ホスト（Pi 5）は、EEPROM（ブートローダ）も更新した。

## 結果

- 上流から取得したのは、1台目の約268MBが中心。2台目以降は0〜1MBで、キャッシュから配信された。
- 全ノードのカーネルが同じ版になった（監視ホストは、EEPROMのみ更新）。

台数が多いほど、キャッシュの効果は大きい。8台だと、体感でも差が出た。

## ハマりどころ

### aptを途中で止めると、次回に失敗する

aptをDL途中で止めると、クライアントの `partial/` に途中ファイルが残る。次回の取得が `Remote end closed connection` で失敗する。該当ファイルを消すと直った。

DL途中で止めただけなら、インストール中じゃないから無害だろうと思っていたが、そんなことはなかった。

しかも、確認と削除をsudoでやろうとしたらできず、仕方なくrootに入って作業した。`partial/` は一般ユーザーが読めないので、`sudo rm .../partial/*` のようにglobを使うと、globがsudoより前に一般ユーザーのシェルで展開されて失敗する。これは定番のハマりどころだ。今回がそれだったかは、確認していない。

### 仮説が外れたときは、数字で切り分ける

失敗の原因を、気分で決めないようにした。ログ、ファイルサイズ、ルールのカウンタで切り分けた。

特に効いたのが、iptablesのカウンタをdigの前後で比較する方法。パケットがどのルールを通ったかが、数字の増減で見える。これでDNATされたあとの `FORWARD` のデフォルトDROPで落ちていると特定できた（未解決の項を参照）。

### 日本語表示だと `Err:` が出ないことがある

ロケールが日本語だと、`Err:` の行が出ないことがあった。`grep -c Err` の件数が当てにならない。`E:` の行で見るのが確実だった。

というか英語統一したほうがいいかもしれない。

やるとしたら別の機会だろう。

## 未解決

GWの仮想IP（VIP）宛てのDNSが、DNAT後の `FORWARD` のDROPで落ちる。

GWはTailscaleやポート転送と同居している。ルールを足せば通せそうだが、他の転送に影響が出ないか見てからにしたい。
というかだるいので後回しにした。

実機での確認が残っているのは、次の3つ。

- Tailscale Split DNS経由の名前解決
- 監視でDNSノードがOKに戻るか
- 新しい端末のDHCPリース

Tailscaleとローカルネットワーク周りはかなりこじれてるのでこちらも次回。

## 所感

ほとんどミートプロキシだったけど何故か楽しいと感じるので引き続きこの趣味は続くと思う。

## 参考

- Debian Project, "Debian 11 Long Term Support reaches end-of-life"（2026-08-31）: <https://www.debian.org/News/2026/20260831.en.html>
- Debian APT team, apt(8) マニュアル（日本語訳、manpages.debian.org）: <https://manpages.debian.org/testing/apt/apt.8.ja.html>
- Eduard Bloch, "Apt-Cacher NG" 公式ページ: <https://www.unix-ag.uni-kl.de/~bloch/acng/>
- jauderho, "HOWTO: Upgrade Raspberry Pi OS from Bullseye to Bookworm"（GitHub Gist、個人による非公式の手順）: <https://gist.github.com/jauderho/6b7d42030e264a135450ecc0ba521bd8>
- Raspberry Pi Ltd, "config.txt"（公式ドキュメント。Bookwormより前は `/boot/` だったという注記がある）: <https://www.raspberrypi.com/documentation/computers/config_txt.html>
- Debian APT team, apt(8) マニュアル（`upgrade` と `full-upgrade` の説明）: <https://manpages.debian.org/unstable/apt/apt.8.en.html>
- Debian APT team, apt-get(8) マニュアル（bookworm版。`upgrade` は未導入パッケージを入れない）: <https://manpages.debian.org/bookworm/apt/apt-get.8>
- Raspberry Pi Ltd, "Bookworm — the new version of Raspberry Pi OS"（公式ニュース。メジャー更新は焼き直しが基本）: <https://www.raspberrypi.com/news/bookworm-the-new-version-of-raspberry-pi-os/>
- Eduard Bloch, "Apt-Cacher-NG User Manual"（公式マニュアル、0.9.1版のPDF。8.3 Access to SSL/TLS remotes）: <https://sources.debian.org/data/main/a/apt-cacher-ng/0.9.1-1~bpo7+1/doc/apt-cacher-ng.pdf>
- Debian Wiki, "AptCacherNg"（HTTPSリポジトリの扱い）: <https://wiki.debian.org/AptCacherNg>
- packagecloud, "Using apt-cacher-ng with SSL/TLS"（PassThroughPatternを設定してもHTTPSはキャッシュされない）: <https://packagecloud.io/blog/using-apt-cacher-ng-with-ssl-tls/>
- Qubes OS Forum, "Cacher(apt-cacher-ng) & Debian template extrepo"（`403 CONNECT denied` の報告）: <https://forum.qubes-os.org/t/cacher-apt-cacher-ng-debian-template-extrepo/29392>
