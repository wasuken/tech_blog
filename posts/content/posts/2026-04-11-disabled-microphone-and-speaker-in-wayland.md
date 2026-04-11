---
title: "WaylandでモニターがマイクとスピーカーとしてOSに認識される問題をWirePlumberで無効化する"
date: 2026-04-11T00:00:00+09:00
draft: false
tags: ["Linux", "Wayland", "PipeWire", "WirePlumber", "ALSA", "ArchLinux"]
---

## 問題

ふとDesktopの画面を見るとモニターがマイクとスピーカーとしてOSに認識されていた。
誤って爆音で音が再生されるリスクが気になったため無効化することにした。
ついでにマイクも有効にする意味がない環境だったので止めた。

純粋な開発PCで動画とかも見ないそこそこ特殊？な環境なので最悪読み込まないなら何でもいい状態。

## 環境

- OS: ArchLinux
- サウンドサーバー: PipeWire + WirePlumber 0.5.14
- GPU: AMD Ryzen（APU）
- 問題のデバイス: `AMD/ATI Raven/Raven2/Fenghuang HDMI/DP Audio Controller`

## 原因

HDMI/DisplayPortには映像だけでなく音声も伝送できる仕様（Audio over HDMI）がある。
LinuxはこれをALSAレベルで別サウンドカードとして認識するため、
PipeWireがそのまま拾ってオーディオデバイスとして公開してしまう。

## 調査

### 認識されているカードを確認

```bash
pactl list cards short
```

```
49      alsa_card.pci-0000_04_00.1      alsa
50      alsa_card.pci-0000_04_00.6      alsa
```

2枚のサウンドカードが認識されている。詳細を確認する。

```bash
pactl list cards | grep -A 30 "alsa_card.pci-0000_04_00"
```

結果を整理すると：

| PCI アドレス | ベンダー | 説明 | 用途 |
|---|---|---|---|
| `0000:04:00.1` | AMD/ATI | Raven HDMI/DP Audio Controller | **モニター側（不要）** |
| `0000:04:00.6` | AMD + Realtek ALC269VB | Ryzen HD Audio Controller | **本物のオンボードサウンド** |

`0000:04:00.1` の `alsa_mixer_name` が `ATI R6xx HDMI` であることからも、
これがHDMI経由のオーディオデバイスだと確定できる。

### 一時的に無効化して動作確認

```bash
pactl set-card-profile 49 off
```

これでモニター側のデバイスがOSから消える。ただし再起動で元に戻る。

## 解決：WirePlumberで永続化

WirePlumber 0.5系ではLuaではなく **`.conf` 形式** で設定を記述する。

```bash
mkdir -p ~/.config/wireplumber/wireplumber.conf.d/
```

```conf
# ~/.config/wireplumber/wireplumber.conf.d/51-disable-hdmi-audio.conf
monitor.alsa.rules = [
  {
    matches = [
      {
        device.name = "alsa_card.pci-0000_04_00.1"
      }
    ]
    actions = {
      update-props = {
        device.disabled = true
      }
    }
  }
]
```

```bash
systemctl --user restart wireplumber
pactl list cards short
```

`alsa_card.pci-0000_04_00.1` が消えていれば成功。

## WirePlumber 0.4系との違い

0.4系ではLuaで記述するのが一般的だった。

```lua
-- 0.4系の書き方（0.5系では動かない）
rule = {
  matches = {
    {
      { "device.name", "=", "alsa_card.pci-0000_04_00.1" },
    },
  },
  apply_properties = {
    ["device.disabled"] = true,
  },
}
table.insert(alsa_monitor.rules, rule)
```

0.5系でLua設定を書いても無視されるため、必ずバージョンを確認してから設定すること。

```bash
wireplumber --version
```

## 結果

- `Dummy output` 表示になるが、これは正常。実際の出力先がないためのフォールバック表示
- マイクデバイスも存在しないため盗聴リスクなし
- 突然爆音が鳴るリスクもなし
- `alsa_card.pci-0000_04_00.6`（Realtek）はそのまま残り、通常のサウンドが使える

## 参考

- [ArchWiki: PipeWire](https://wiki.archlinux.org/title/PipeWire)
