---
title: "Raspberry Pi 2台構成のWiFi APでSSIDが起動時に出ない問題を解決した"
date: 2026-05-06T18:00:00+09:00
draft: false
tags: ["Raspberry Pi", "hostapd", "NetworkManager", "Linux", "ネットワーク", "WiFi"]
categories: ["インフラ"]
description: "Raspberry Pi 2台でESS構成（同一SSID・別チャンネル）のWiFi APを組んだところ、OS起動時にSSIDが出ない問題が発生。NetworkManagerとhostapdの起動順序が原因だったのでsystemdのoverride.confで解決した。"
---

## 構成

自宅NWの冗長化目的で Raspberry Pi を2台使い、WiFiアクセスポイントを組んでいる。

| ホスト名 | チャンネル | SSID |
|---|---|---|
| pi-1 | 1 | your-ssid |
| pi-2 | 11 | your-ssid |

同一SSIDで別チャンネルにする構成は **ESS（Extended Service Set）** と呼ばれ、クライアントが自動的に電波の強い方に接続する。WPA2-PSKで普通に成立する冗長化構成。

ブリッジ構成で bridge interface（`br0`）にWiFiインタフェース（`wlan0`）を参加させ、hostapdで電波を飛ばしている。

---

## 問題

OS再起動後、SSIDが片方または両方スキャンに出ない。

- `systemctl status hostapd` は `active (running)`
- エラーログも一見出ていない
- hostapdを手動で再起動すると出る

---

## 調査

### チャンネルの干渉を疑う

最初にhostapd.confのチャンネル設定を確認した。

2.4GHz帯で実際に干渉しないチャンネルは **1、6、11** の3つだけ。元の設定は6と7だったため、ほぼ完全に干渉していた。

```
ch6  --|--
ch7    --|--   ← 隣接していてほぼ被る
```

チャンネルを1と11に変更した。

```ini
# pi-1
channel=1

# pi-2
channel=11
```

### BSSIDレベルで電波を確認

```bash
sudo iw dev wlan0 scan | grep -E "BSS |SSID|freq|signal"
```

```
BSS xx:xx:xx:xx:xx:xx(on wlan0)
        freq: 2462
        signal: -15.00 dBm
        SSID: your-ssid
BSS xx:xx:xx:xx:xx:xx(on wlan0)
        freq: 2412
        signal: -68.00 dBm
        SSID: your-ssid
```

両台ともビーコンは出ていた。ESSとして動作は成立している。

### power_saveを疑う

APなのにpower_saveが有効になっている可能性を確認した。

```bash
sudo iw dev wlan0 get power_save
```

hostapdが `wlan0` を掴んでいるため、設定変更にはhostapd停止が必要：

```bash
sudo systemctl stop hostapd
sudo iw dev wlan0 set power_save off
sudo systemctl start hostapd
```

ただし再起動で戻るため、永続化はsystemdのoverride.confで対応：

```ini
# /etc/systemd/system/hostapd.service.d/override.conf
[Service]
ExecStartPost=/sbin/iw dev wlan0 set power_save off
```

これだけでは問題は解決しなかった。

### beacon_intを疑う

元の設定は `beacon_int=200`（デフォルトは100ms）。ビーコン間隔が長いと、起動直後のスキャンでSSIDを見逃す可能性がある。

```ini
beacon_int=100
```

に戻したが、問題は解決しなかった。

### journalctlで起動ログを確認

```bash
sudo journalctl -u hostapd -b 0
```

pi-2のログに以下が出ていた：

```
hostapd: nl80211: Failed to add the bridge interface br0: File exists
hostapd: nl80211 driver initialization failed.
hostapd: wlan0: interface state UNINITIALIZED->DISABLED
hostapd: wlan0: AP-DISABLED
hostapd: wlan0: CTRL-EVENT-TERMINATING
hostapd.service: Failed with result 'exit-code'.
```

その後：

```
hostapd.service: Scheduled restart job, restart counter is at 1.
hostapd: wlan0: AP-ENABLED
```

**原因はここにあった。** `Restart=on-failure` がセーフティネットになっていただけで、初回起動は失敗していた。

### 起動順序を確認

```bash
sudo systemctl show hostapd | grep -E "After|Wants|Requires"
```

pi-1とpi-2で差異があった：

```
# pi-1
After=sys-subsystem-net-devices-wlan0.device network.target networking.service ...

# pi-2
After=network.target   ← wlan0.deviceもnetworking.serviceも待っていない
```

pi-2は `network.target` しか待っていなかった。

### ブリッジの管理主体を確認

```bash
ls /etc/NetworkManager/system-connections/
# br0.nmconnection  bridge-slave-eth0.nmconnection
```

br0は **NetworkManager** が管理していた。

---

## 原因

起動フローが以下の順序になっていた：

```
1. OS起動
2. NetworkManager が br0 を作成（非同期）
3. hostapd が起動 → br0 がまだ中途半端な状態
4. "Failed to add the bridge interface br0: File exists" で失敗
5. Restart=on-failure で2秒後に再起動
6. 今度はbr0が正常になっているので成功
```

`File exists` エラーは「br0が存在しない」ではなく「br0が中途半端な状態で既に存在している」ことを示している。hostapdが `wlan0` をbr0に追加しようとした時に競合が発生していた。

`systemctl status` が `active` に見えるのは `Restart=on-failure` が自動回復しているためで、SSIDが出るのが遅延していた。

---

## 解決策

両台に `systemd` の override.conf を作成し、`network-online.target` を待つようにする。

```bash
sudo mkdir -p /etc/systemd/system/hostapd.service.d/
sudo nano /etc/systemd/system/hostapd.service.d/override.conf
```

```ini
[Unit]
Wants=sys-subsystem-net-devices-wlan0.device network-online.target
After=sys-subsystem-net-devices-wlan0.device network-online.target
```

```bash
sudo systemctl daemon-reload
sudo reboot
```

### network-online.targetが有効か確認

```bash
sudo systemctl is-enabled NetworkManager-wait-online.service
```

`enabled` になっていること。`disabled` の場合は `network-online.target` が機能しないので有効化する：

```bash
sudo systemctl enable NetworkManager-wait-online.service
```

---

## network-online.target と network.target の違い

| target | 意味 |
|---|---|
| `network.target` | ネットワークスタックの起動完了。インターフェースがオンラインかは問わない |
| `network-online.target` | 少なくとも1つのインターフェースがオンラインになった状態 |

NetworkManagerが管理するbr0のような仮想インターフェースを依存先にする場合は `network-online.target` が適切。

---

## 補足：hostapd.confの注意点

### TKIPはCCMPに統一する

```ini
# 非推奨
wpa_pairwise=TKIP

# 推奨
wpa_pairwise=CCMP
rsn_pairwise=CCMP
```

TKIPはWPA2と組み合わせた場合に一部クライアントから拒否されることがある。

### チャンネルは1・6・11を使う

2.4GHz帯で干渉しないチャンネルの組み合わせは **1と6**、**1と11**、**6と11** のみ。隣接チャンネルは必ず干渉する。

---

## まとめ

| 項目 | 内容 |
|---|---|
| 原因 | NetworkManagerによるbr0初期化完了前にhostapdが起動 |
| 症状 | `Restart=on-failure` で自動回復するため `active` に見えるが遅延あり |
| 対処 | `network-online.target` をAfter/Wantsに追加 |
| 対象 | NetworkManagerがブリッジを管理している環境全般 |

`systemctl status` が `active` でも、`journalctl -u hostapd -b 0` で起動ログを必ず確認すること。
