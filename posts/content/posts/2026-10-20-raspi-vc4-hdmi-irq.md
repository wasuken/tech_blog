---
title: "Raspberry Piで見かけた「irq/173-vc4 hdmi hpd」は何者か"
date: 2026-10-20T09:00:00+09:00
draft: false
ai_assisted: true
tags: ["Linux", "kernel", "Raspberry Pi"]
categories: ["技術"]
description: "Raspberry Piのプロセス一覧に出てくるvc4 hdmi系のカーネルスレッドについて、調べた範囲でまとめたメモ"
---

## はじめに

`htop` を眺めていたらこんな名前のスレッドが並んでいた。

```text
irq/173-vc4 hdmi hpd connected
irq/174-vc4 hdmi hpd disconnected
irq/175-vc4 hdmi cec rx
irq/176-vc4 hdmi cec tx
```

HDMIは挿していないのに connected も disconnected もいるので、少し気になって調べた。

**調べた範囲では、HDMIドライバが作る割り込み処理用のカーネルスレッドで、特に異常なものではなさそう。** 

以下はそのメモ

なお番号は環境によって違う。

## 名前の読み方

| 部分 | 意味 |
|---|---|
| `irq/173` | IRQ番号173の割り込みを処理するカーネルスレッド |
| `vc4` | Raspberry Piのディスプレイ周りを担当するLinuxのDRMドライバ |
| `hpd` | Hot Plug Detect。HDMIの抜き挿しの検知 |
| `cec` | HDMI経由でテレビなどを操作する仕組み。`rx`が受信、`tx`が送信 |

## なぜHDMIを挿していなくてもいるのか

カーネルのソースを見ると、vc4のHDMIドライバは初期化のときに `devm_request_threaded_irq()` で割り込みを登録している。このとき渡す名前の文字列が、そのままスレッド名（`irq/番号-名前`）になる。

以下はLinux mainlineのコードをさらに簡略化したもの。

Raspberry Pi OSのカーネルは独自のツリーなので細部は違うかもしれない。

```c
/* drivers/gpu/drm/vc4/vc4_hdmi.c より */
ret = devm_request_threaded_irq(&pdev->dev, hpd, NULL,
                                vc4_hdmi_hpd_irq_thread, IRQF_ONESHOT,
                                "vc4 hdmi hpd connected", vc4_hdmi);
```

つまりスレッドは **HDMIが挿さったとき** ではなく **ドライバが割り込みの待ち受けを登録したとき** に作られる。

接続用と切断用で割り込みが別々に登録されるので、`connected` と `disconnected` が両方並ぶ。

**接続と切断を繰り返しているわけではなく それぞれ待機しているだけ** という理解でよさそう。

なお、この登録はコード上 `external_irq_controller` が有効なSoC（BCM2711＝Pi 4系、BCM2712＝Pi 5）の分岐の中にある。

Pi 3以前では、見え方が違うかもしれない（手元では未確認）。

## 放置でよいのか

シンプルに手元で瞬間的に見た限りではCPU使用率は0.0%だった。

そもそも割り込みが来るまで待っているだけなので、通常は負荷にならないはずなので放置でよいだろう。

ヘッドレス運用でも普通に出る表示だと思う。

消したい場合はKMSドライバの設定を触ることになりそうだが、画面出力まわりにも影響するので

プロセス一覧をすっきりさせる目的ならやらなくていいと思う。

## 確認方法

```bash
grep -Ei 'vc4|hpd|cec' /proc/interrupts
```

割り込み名や回数が見られる。

経過時間とCPU時間を並べたいなら次のコマンドが使える。

```bash
ps -eo pid,etime,time,args | grep '[i]rq/'
```

## 参考

- [Linux mainline — drivers/gpu/drm/vc4/vc4_hdmi.c](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/vc4/vc4_hdmi.c)
- [Linux Kernel Documentation — Linux generic IRQ handling](https://docs.kernel.org/core-api/genericirq.html)
- [drm/vc4: hdmi: Rely on interrupts to handle hotplug（Maxime Ripard, LKML, 2021-05-24）](https://lkml.iu.edu/hypermail/linux/kernel/2105.3/00446.html)
