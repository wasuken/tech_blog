---
title: "Linuxの起動フローを整理する - UEFI/BIOSからinitまで"
date: 2026-02-28T00:00:00+09:00
draft: false
tags: ["linux", "kernel", "boot", "infrastructure", "sre"]
categories: ["infrastructure"]
description: "UEFI/BIOSからブートローダー、カーネル、initまでのLinux起動フローをSRE視点で整理する"
---

## はじめに


[https://info.nikkeibp.co.jp/media/LIN/atcl/books/070900046/:embed:cite]

上記の技術書を読んでいて、ブートローダとLinuxの初期スタート時の役割とか順番がいまいち掴めなかったので生成AIや他の記事など別軸から調べ直してまとめた。

---

## 起動フロー全体像

```
UEFI/BIOS
  ↓  POST（ハードウェア初期化）、ブートデバイス選択
ブートローダー（GRUB等）
  ↓  /boot/vmlinuz（カーネルイメージ）をメモリに展開
  ↓  /boot/initramfs をメモリに展開
カーネル起動
  ↓  initramfsを一時的な / としてマウント
  ↓  ドライバ読み込み、本物のrootデバイスを認識
  ↓  本物のroot FSをマウント（switch_root）
/sbin/init（systemd）に移譲
```

---

## 各フェーズの詳細

### 1. UEFI/BIOS

起動の最初はUEFI（または旧来のBIOS）が担う。

- **POST（Power-On Self Test）**: メモリ、CPU、周辺デバイスの初期化
- ブートデバイスの選択（NVMe, SSD, PXEなど）
- UEFIの場合はEFIパーティション（ESP）から `.efi` ファイルを直接実行できる

UEFIとBIOSの大きな違いとして、UEFIはGPTディスクのネイティブサポートや、セキュアブートの仕組みを持つ。

### 2. ブートローダー（GRUB2等）

UEFI/BIOSからブートローダーに制御が渡る。
代表的なものはGRUB2で、設定ファイルは `/boot/grub/grub.cfg` にある。

ブートローダーの役割は**シンプル**で、以下の2点だけ：

1. カーネルイメージ（`vmlinuz`）をメモリに展開する
2. initramfs（`initramfs-*.img`）をメモリに展開する

```bash
# /boot 以下の典型的な構成
$ ls /boot/
grub/
initramfs-6.1.0-28-amd64.img
vmlinuz-6.1.0-28-amd64
```

ブートローダー自身は**ルートFSのマウントをしない**。あくまでカーネルとinitramfsをメモリに置いて制御を渡すだけ。

### 3. カーネル起動とinitramfs

ここが一番誤解されやすいフェーズ。

カーネルが起動すると、まず**initramfs（Initial RAM Filesystem）**を一時的なルート（`/`）としてマウントする。

**なぜinitramfsが必要か？**

カーネル本体はコンパクトに保つ設計になっており、NVMeやLVMやLUKS（暗号化）といった本物のディスクにアクセスするためのドライバを、起動時に動的にロードする必要がある。
initramfsはそのためのミニマルな環境を提供する。

```
initramfs の中身（概略）
  /init        → 起動スクリプト
  /lib/modules → カーネルモジュール（ドライバ）
  /bin, /sbin  → busybox等の最低限のコマンド群
```

処理の流れ：

1. initramfs内の `/init` スクリプトが実行される
2. 必要なカーネルモジュール（ドライバ）をロード
3. 本物のrootデバイス（`/dev/nvme0n1p2` 等）を認識
4. 本物のroot FSをマウント
5. `switch_root` で本物の `/` に切り替え

### 4. /sbin/init（systemd）への移譲

`switch_root` が完了すると、カーネルは `/sbin/init` を PID 1 として起動して移譲完了。

現代のLinuxディストリビューションでは、`/sbin/init` は `systemd` へのシンボリックリンクになっている。

```bash
$ ls -la /sbin/init
lrwxrwxrwx 1 root root 20 /sbin/init -> /lib/systemd/systemd
```

systemdはここからユニットファイルに従ってサービスを順次起動していく。

---

## よくある混乱ポイントの整理

| 疑問 | 答え |
|------|------|
| rootFSをマウントするのは誰？ | カーネル（initramfs経由） |
| ブートローダーは何をする？ | カーネルとinitramfsをメモリに置くだけ |
| initramfsが必要な理由は？ | カーネルが本物のディスクドライバをロードするための踏み台 |
| `/sbin/init` の正体は？ | 現代ではほぼsystemdへのシンボリックリンク |

---

## 知っておくべきこと

**カーネルパニック時の読み方**

起動フローを把握していると、カーネルパニックのメッセージがどのフェーズで発生したかを特定しやすくなる。
`VFS: Unable to mount root fs` のようなエラーはinitramfsフェーズの問題、`Kernel panic - not syncing: No working init found` はinit移譲の失敗を示す。

**GRUBレスキュー**

ブートローダーの設定が壊れた場合、GRUBのレスキューモードから手動でカーネルとinitramfsを指定して起動できる。

```bash
# GRUBレスキューモードでの手動起動例
grub> set root=(hd0,gpt2)
grub> linux /boot/vmlinuz root=/dev/nvme0n1p2
grub> initrd /boot/initramfs.img
grub> boot
```

---

## 参考

- [Linux Kernel Documentation - initrd](https://www.kernel.org/doc/html/latest/admin-guide/initrd.html)
- [Arch Wiki - Arch boot process](https://wiki.archlinux.org/title/Arch_boot_process)
- [freedesktop.org - systemd](https://www.freedesktop.org/wiki/Software/systemd/)
- [GNU GRUB Manual](https://www.gnu.org/software/grub/manual/grub/grub.html)
