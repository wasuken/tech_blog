---
title: "ArchLinuxのThunarでWalkmanのFSを開く"
date: 2026-04-04T17:00:00+09:00
draft: false
tags: ["Linux", "ArchLinux", "MTP", "オーディオ"]
---

## 背景

手持ちのWalkmanをLinux（Arch Linux）環境で活用したいと考えた。
単に音楽を聴くだけでなく、PCのファイルを転送したり、時にはPCの音を高音質で鳴らすオーディオインターフェースとして使いこなすのが目的だ。

ドキュメントを読む限り、最近のデバイスはMTP（Media Transfer Protocol）に対応しており、Linuxでも標準的なツールで扱えるはずだ。

### 環境

- **OS**: Arch Linux
- **File Manager**: Thunar
- **Device**: Walkman (MTP/USB DAC対応モデル)
- **Tools**: `usbutils`, `gvfs-mtp`, `libmtp`, `jmtpfs`

---

## ThunarでWalkmanのFSが見えない

WalkmanをUSBケーブルでPCに接続し、Thunarを開いたがサイドバーには何も表示されない。

まず物理的な接続を確認しようと `lsusb` を叩いたところ、コマンド自体が入っていなかった。

```bash
sudo pacman -S usbutils
```

改めて確認する。

```bash
$ lsusb
Bus 001 Device 008: ID 054c:0c2f Sony Corp. Walkman
```

デバイス自体はUSBレベルでは認識されている。`fdisk -l` にブロックデバイスとして出てこないのはMTPなので当然だ。

---

## 原因：MTP用ライブラリが未インストール

`gvfs-mtp` と `libmtp` が入っていないのが原因だった。

```bash
sudo pacman -S gvfs-mtp libmtp

# Thunarを再起動して反映
thunar -q
```

これでThunarのサイドバーにWalkmanが表示され、GUIでファイルをコピーできるようになった。

---

## 補足：USB DACモードとは

調査中に「DACモードでなければ動かないのか？」と気になって調べたのでここにまとめておく。

USB DACモードとは、デバイスを「ストレージ」としてではなく、**「USBオーディオデバイス」**としてPCに認識させるモードだ。ファイル転送には使えない。

| モード | PCからの見え方 | 用途 |
|---|---|---|
| MTP / MSC | ストレージ | ファイル転送 |
| USB DAC | オーディオデバイス | PC音声出力 |

Walkman側の設定でどちらのモードになっているかは確認しておく必要がある。

---

## GUIで認識しない場合の手動マウント

`gvfs-mtp` を入れてもThunarに出ない場合は `jmtpfs` で手動マウントできる。

```bash
# jmtpfsをインストール（AUR）
yay -S jmtpfs

# マウントポイントを作成してマウント
mkdir -p ~/mnt/walkman
jmtpfs ~/mnt/walkman

# アンマウント
fusermount -u ~/mnt/walkman
```

---

## まとめ

ThunarでMTPデバイスのFSを参照するには `gvfs-mtp` と `libmtp` が必要だ。`lsusb` でデバイスが見えていても、これらがなければファイルマネージャーには出てこない。

接続が認識されているかどうかの切り分けは以下の順で行うとよい。

1. `lsusb` でUSBレベルの認識を確認（`usbutils` が必要）
2. Walkman側のUSBモードがファイル転送になっているか確認
3. `gvfs-mtp` / `libmtp` のインストール
4. それでも駄目なら `jmtpfs` で手動マウント
