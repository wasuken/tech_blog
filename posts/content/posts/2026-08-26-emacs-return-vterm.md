---
title: "日記 EatからVtermへ出戻りした話とかeglot(rust)動かない問題とか"
date: 2026-08-26T20:50:00+09:00
draft: false
ai_assisted: true
tags: ["Emacs", "vterm", "eat", "Rust", "RustAnalyzer"]
categories: ["技術"]
description: "EatからVtermに移行してみたが、結局vtermに出戻りすることにした経緯と設定変更のメモ"
---

## TL;DR

- 以前vtermのC-kまわりの挙動を追いかけていて、その流れでEatへ移行していた
- 今回もう一度vtermの素の設定を試したところ、**C-kは特別な回避策なしで普通に動いた**
- Eatは実運用してみると自分のワークフローと噛み合わない部分がいくつかあり、結局vtermに出戻りすることにした
- 副次的にEglotでRustのサーバーが起動しない問題も出ていたので、原因と対処もついでにメモしておく

## 経緯

もともと[vtermのC-kバインドを追いかけていたらEatに移行することになった話](https://techblog.wasutech.dev/posts/emacs-vterm-to-eat/)（wasutech, 2026-07-04）でvtermからEatへ移行していた。この記事は逆に、そのEatからvtermへ戻した記録になる。

ちなみに、上の記事内でEat移行のきっかけとしてリンクされていたのが、Ki_chi氏の[EmacsのターミナルエミュレーターをvtermからEatに移行しました](https://ki-chi.jp/posts/switch-from-vterm-to-emacs-eat/)（Ki_chi@Blog, 2026-01-29）。

## 削除した設定

Eatのuse-packageブロック一式を削除した。内容としては、straight経由でのcodebergリポジトリ指定、`eat-semi-char-non-bound-keys`の除外リスト編集、`eat-reload`呼び出しなど。

## 復元したvterm設定

元記事内で「移行前の設定」として紹介されていたvterm用のuse-packageブロックをベースに復元した。

```elisp
(defun my/vterm-send-C-k ()
  "Send C-k to vterm terminal."
  (interactive)
  (let ((inhibit-read-only t))
    (vterm-send-key "k" nil nil t)))

(use-package vterm
  :straight t
  :bind (:map vterm-mode-map
              ("C-h" . vterm--self-insert)
              ("C-k" . my/vterm-send-C-k))
  :bind (("C-c v" . vterm))
  :config
  (setq vterm-term-environment-variable "xterm-256color"))
```

`defun`を`use-package`より前に置いているのは、Emacs Lispが上から評価される言語だから。先に定義しておかないと、`use-package`の`:bind`が参照する段階で未定義エラーになる。

`my/vterm-send-C-k`自体は、元記事の著者本人も根本原因は特定できなかったと明記している回避策で、いわば対症療法である。

実際に設定を戻してみたところ、**C-kは回避策なしの素の状態で普通に動いた**。以前この問題を追いかけていた時点との違い（Emacsのバージョン、vtermのバージョン、あるいは単に環境差）は特定できていないので、なぜ動くようになったのかは正直わかっていない。ただ動いている以上、今のところ回避策は入れずに使っている。

## 別件: eglot(rust-analyzer)が起動しない

vterm/eatの話とは別件だが、ちょうど同じタイミングでEglotのRustサーバー起動に失敗する問題が出た。

原因は単純で、`rustc`と`rust-analyzer`はrustupのコンポーネントとして別扱いになっており、公式toolchainをインストールしただけでは`rust-analyzer`は入らない。個別にインストールする必要がある。

```sh
rustup component add rust-analyzer
```

これで解決した。

## 出戻りマップ

| 項目 | Eat | Vterm |
|---|---|---|
| 未Git管理ディレクトリを開いたとき | ディレクトリを聞かれてモヤっとする | 特に気にならない |
| レイアウト | 崩れることがしばしばあった | 安定している |
| 実装 | Emacs Lispのみで完結（要確認） | libvterm（C）に依存 |
| C-k | 今回は未検証 | 素の設定で動作した |

## 所感

Eatは実装がEmacs Lisp単体で完結しているというのは魅力的で、パッケージとしての完成度も高いと思う。とはいえ、自分の使い方だと未Git管理のディレクトリを開こうとしたときにディレクトリを聞かれたり、レイアウトがよく崩れたりすることがしばしばあり、実際に使ってみるとかえって煩わしさの方が勝ってしまった。Emacsで完結するツールとしては素晴らしいパッケージだと思うが、結果的には私にはvtermの方が合っていたので、恥ずかしながら出戻りすることになった。とはいえ、また別のターミナルエミュレータを試すかもしれないので、結論は一旦保留にしておく。今はvtermを使う。

## 参考

- [vtermのC-kバインドを追いかけていたらEatに移行することになった話](https://techblog.wasutech.dev/posts/emacs-vterm-to-eat/)（wasutech, 2026-07-04）
