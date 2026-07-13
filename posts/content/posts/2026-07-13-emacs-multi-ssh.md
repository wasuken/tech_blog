---
title: "EmacsのTRAMPで多段SSH越しにファイルを開いた話"
date: 2026-07-13T21:00:00+09:00
draft: false
ai_assisted: true
tags: ["Emacs", "TRAMP", "SSH", "ホームラボ"]
categories: ["tech"]
description: "A→B→CとSSHで踏み台越しにファイルを開きたくてTRAMPを触ったら、記憶していた構文が古くて普通にハマった話。"
---

## きっかけ

作業機Aで書いたEmacsの設定はそのままに、B経由でCにあるファイルを直接開きたくなった。CにEmacsを立てるほどでもないし、B自体も経由するだけの踏み台。よくある構成だと思う。

「TRAMPで多段SSHすれば一発で開けるはず」という認識だけはあったので、記憶を頼りに適当な構文を書いたら普通に動かなかった。

## 最初に書いた(間違った)構文

```
C-x C-f /ssh:userB@B:/ssh:userC@C:/path/to/file RET
```

コロン区切りでリモートパスを入れ子にすれば多段になるだろう、という思い込みで書いたが、これは単に存在しないパスとして扱われて開けない。

## 正しい構文: パイプ区切り

調べ直すと、現行のTRAMP(Ad-hoc multi-hops)では各ホップを `|` で連結する仕様になっていた。

各プロキシはファイル名部分を除いたリモートホスト指定と同じ構文で指定し、ホップごとに `|` で区切って、起点ホストから最終目的地までを連結する、という説明になっている[^tramp-manual]。

```
C-x C-f /ssh:userB@B|ssh:userC@C:/path/to/file RET
```

パスの `:` は最後のホップにだけ付く。それ以前のホップは `|` で単純に繋げるだけでよかった。

`dired` でディレクトリブラウズしたい場合も同じ構文でいける。

```
C-x d /ssh:userB@B|ssh:userC@C:/path/to/dir RET
```

## 一度使うと短縮形が使えるようになる

TRAMPはこのアドホックな多段定義を、そのEmacsセッション中は `tramp-default-proxies-alist` に一時的なレコードとして追加する。そのため同じセッション内であれば、以降は `/ssh:you@remotehost:/path` という単純な形式だけで同じリモートホストに再接続できるようになる[^tramp-manual]。

セッションをまたいで使い回したい場合は `tramp-save-ad-hoc-proxies` を非nilにしておけば、設定ファイル側に多段定義そのものが保存される[^tramp-manual]。

## ついでにsudoも同じノリで組める

C上でrootが必要な作業をしたい場合、`sudo` を最後のホップとして追加するだけでよい。

```
C-x C-f /ssh:userB@B|ssh:userC@C|sudo:root@C:/path/to/file RET
```

`su`、`sudo`、`doas`、`run0` のようなメソッドを別ホスト上で実行したい場合、先頭にsshなどのメソッドを組み合わせて使う。つまりTRAMPはまず非管理者権限でそのホストに接続し、そのあとでそのホスト上で管理者権限に切り替える、という2段階の動作になっている[^tramp-manual]。

現在開いているバッファをそのままsudo権限で開き直したいだけなら、`tramp-revert-buffer-with-sudo` というコマンドが用意されている[^tramp-manual2]。ファイルを開き直すたびにパスを書き直さなくていいので、これが一番出番が多いかもしれない。

```
M-x tramp-revert-buffer-with-sudo RET
```

## まとめ

TRAMPの多段SSHは、以前は `tramp-default-proxies-alist` を事前に設定しておく必要があったが、Emacs 24以降は設定なしでその場でパイプ区切りの多段パスを書くだけで通るようになっている[^tramp-wikemacs]。この「昔の記憶のまま新しい仕様を疑わなかった」せいで無駄にハマったので、次から多段SSH系のパスを書くときは一旦マニュアルを見に行くことにする。

[^tramp-manual]: [Ad-hoc multi-hops (TRAMP User Manual)](https://www.gnu.org/software/emacs/manual/html_node/tramp/Ad_002dhoc-multi_002dhops.html)
[^tramp-manual2]: [TRAMP 2.8.1 User Manual](https://www.gnu.org/software/tramp/)
[^tramp-wikemacs]: [TRAMP - WikEmacs](https://wikemacs.org/wiki/TRAMP)
