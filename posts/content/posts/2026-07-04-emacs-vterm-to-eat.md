---
title: "vtermのC-kバインドを追いかけていたらEatに移行することになった話"
date: 2026-07-04T18:00:00+09:00
draft: false
ai_assisted: true
tags: ["emacs", "vterm", "eat", "terminal", "elisp", "straight.el"]
categories: ["tech"]
description: "EmacsのvtermでC-kが動かず調査していたら、原因は特定できないままEatへの移行を決めた。移行の過程でstraight.elのデフォルトレシピが原因のterminfoエラーと、Eat独自のキー除外リストの扱いにハマったのでまとめる。"
---

# vterm -> eatした理由

vtermでC-hとかは動いたけどC-kがうまく動かなかった。

色々ガチャガチャやってたけどだるくなったときにeatというものを知ったので試してみたところ、

ターミナルとして要求していたことをほぼ達成できたため、移行することにした。

[EmacsのターミナルエミュレーターをvtermからEatに移行しました | Ki_chi@Blog](https://ki-chi.jp/posts/switch-from-vterm-to-emacs-eat/)

今回のEat移行の設定はこの記事を参考にした。本当に感謝。

また、これまで利用していたvtermについても感謝を伝えたい。恐らく要求されることはできたとは思うが、私の熱量が足りないためにそれを実現できなかったと見ている。

## 発端: vtermのC-kが動かない

vtermで以下のような設定を書いていた。

```elisp
(defun my/vterm-send-C-k ()
  "Send C-k to vterm terminal."
  (interactive)
  (let ((inhibit-read-only t))
    (vterm-send-key "k" nil nil t)))

(use-package vterm
  :bind (:map vterm-mode-map
              ("C-h" . vterm--self-insert)
              ("C-k" . my/vterm-send-C-k))
  :config
  (setq vterm-term-environment-variable "xterm-256color"))
```

`C-h`は狙い通り動くのに`C-k`だけ`Buffer is read-only`エラーで弾かれる。

最初は「`vterm-keymap-exceptions`のデフォルトに`C-k`が含まれているせいだろう」と当たりをつけたが、実際にvtermのソースを確認したところこれは誤りだった。デフォルトの除外リストは以下で、`C-k`は含まれていない。

```elisp
'("C-c" "C-x" "C-u" "C-g" "C-h" "C-l" "M-x" "M-o" "C-y" "M-y")
```

さらに`vterm-send-key`自体の実装を見ると、関数内部で`inhibit-read-only`を自前でletしている。つまり`my/vterm-send-C-k`が本当に呼ばれているなら、read-onlyエラーはそもそも起きようがない。

ということは、`C-k`は自分が定義した関数にディスパッチされておらず、別の何か（`kill-line`のような通常のEmacs編集コマンドなど）が割り込んでいることになる。`C-h k C-k`で実際の解決結果を確認すれば犯人は分かるはずだが、ここまで調べた時点で「そもそもvtermをやめてEatに乗り換える」という選択肢が視野に入ってきたので、根本原因の特定は一旦保留にした。

## なぜEatなのか

そもそもvtermのキーマップあれこれで詰まってしまったのがすべてではあるが、後付の理由もあって、

Eatは実装がすべてEmacs Lispで完結している点は素晴らしいと思う。コンパイル済みモジュールに依存しないので初期でcmakeを要求されたりしない。所詮cmakeかつlibvtermのため、ほとんどの環境で動作するとは思うが、個人的にはElispオンリーというのは少し惹かれた。

## 逆に怖いところ

[GitHub - akermu/emacs-libvterm: Emacs libvterm integration · GitHub](https://github.com/akermu/emacs-libvterm)

[akib/emacs-eat: Emulate A Terminal, in a region, in a buffer and in Eshell - Codeberg.org](https://codeberg.org/akib/emacs-eat)

vtermは先週もコミットが入っている一方、eatは去年当たりから止まっている。

いちおうIssueを見てみると

[#237 - Is this project abandoned? - akib/emacs-eat - Codeberg.org](https://codeberg.org/akib/emacs-eat/issues/237)

作者は生存が確認できており、どうにも忙しいみたいだ。

人に寄ってはこれらの情報もeatを利用する際には小さくない要素となるだろう。

## インストールで詰まった話

use-packageで以下のように書いて導入を試みた。

```elisp
(use-package eat
  :ensure t
  :bind ("C-c v" . eat-project-other-window)
  :config
  (setq vterm-term-environment-variable "xterm-256color"))
```

初回セットアップとして案内されている`M-x eat-compile-terminfo`を実行したところ、次のエラーで止まった。

```
Eat not installed properly: Terminfo source file not found
```

Eatのソースを確認すると、`eat-compile-terminfo`は「実際にロードされた`eat.el`が置かれているディレクトリ」から`eat.ti`というファイルを探しにいく実装になっている。つまりこのエラーは、パッケージとして`.el`ファイル以外（`.ti`やterminfo関連ディレクトリ）が正しく取得されていないことを意味する。

自分のdotfilesリポジトリを確認したところ、`manager.el`で`straight-use-package-by-default t`を設定していた。つまり`use-package :ensure t`は実質straight.el経由のインストールになっていて、straight.elのデフォルトレシピでは`*.ti`のような非標準拡張子のファイルまでは拾いきれていなかった、というのが原因だった。

対処は、`eat`のレシピだけ明示的に`:files`指定で上書きすることだった。

```elisp
(use-package eat
  :straight (eat :type git :host codeberg :repo "akib/emacs-eat"
                  :files ("*.el" ("term" "term/*.el") "*.texi" "*.ti"
                          ("terminfo/e" "terminfo/e/*")
                          ("terminfo/65" "terminfo/65/*")
                          ("integration" "integration/*")
                          (:exclude ".dir-locals.el" "*-tests.el")))
  :bind ("C-c v" . eat-project-other-window))
```

古いビルドが`straight`配下に残っていたので、一度`straight/build/eat`と`straight/repos/eat`を消してから再インストールし、無事`eat-compile-terminfo`が通った。

## 移行後: C-kは直ったがC-hが効かない

Eatに切り替えた結果、`C-l`はターミナル側のクリア処理としてそのまま動作し、懸案だった`C-k`も特に設定を足さずに動くようになった。vterm側の根本原因は結局特定しないままだったが、実用上は解決した形になる。

一方で`C-h`が反応しない。これはEatの仕様で、デフォルトの入力モードである"semi-char"モードには、Emacs側に処理を譲るキーの除外リストがあり、`C-h`はそこに含まれている。vtermの`vterm-keymap-exceptions`と発想は同じだが、Eat側の変数名は`eat-semi-char-non-bound-keys`で、扱い方にクセがある。

その場しのぎなら`C-q`（次の1キーをそのままターミナルに送る）を使えばいい。

```
C-q C-h
```

恒久的に直すなら、除外リストからキーを取り除いた上で、明示的にキーマップの再構築とEatの再読み込みを行う必要がある。単に`setq`しただけでは反映されない。

```elisp
(use-package eat
  :straight (eat :type git :host codeberg :repo "akib/emacs-eat"
                  :files ("*.el" ("term" "term/*.el") "*.texi" "*.ti"
                          ("terminfo/e" "terminfo/e/*")
                          ("terminfo/65" "terminfo/65/*")
                          ("integration" "integration/*")
                          (:exclude ".dir-locals.el" "*-tests.el")))
  :bind ("C-c v" . eat-project-other-window)
  :config
  (setq eat-semi-char-non-bound-keys
        (delete [?\C-h] eat-semi-char-non-bound-keys))
  (eat-update-semi-char-mode-map)
  (eat-reload))
```

`"C-h"`のような文字列ではなく`[?\C-h]`というベクタ表記で指定する必要がある点も、vtermの`vterm-keymap-exceptions`とは違うところで少しハマりポイントだった。

## ターミナルバッファの文字列をコピーしたいとき

Eatに乗り換えてから地味に困ったのが、ターミナル内に表示されている文字列のコピー方法だった。semi-charモードのままだとキー入力がそのままシェルに送られてしまうので、普通にリージョン選択して`M-w`、というわけにはいかない。

手順としては以下になる。

1. `C-c C-e`で"emacs"モードに切り替える。入力送信が止まり、read-onlyな普通のバッファとして扱えるようになる
2. 通常のEmacsコマンドでリージョンを選択し、`M-w`（`kill-ring-save`）でkill-ringにコピー
3. 用が済んだら`C-c C-j`で"semi-char"モードに戻す

コピーしたテキストは`C-y`（`eat-yank`）でEatバッファにも、他の通常のバッファにも貼り付けられる。

一点注意点として、"emacs"モード中も`C-c C-k`だけは特別扱いでプロセスをkillする動作になっている。vtermのcopy-modeとは違い緊急停止用のキーが生きたままなので、選択中に誤って押さないようにしたい。

## まとめ

- vtermの`C-k`問題は、`vterm-keymap-exceptions`のデフォルトに`C-k`は含まれておらず、`vterm-send-key`自体もread-onlyエラーを起こさない実装だったため、根本原因は結局特定できなかった
- Eatへの移行はCモジュールのビルド依存を切り離せる点が動機になった
- `straight-use-package-by-default t`な環境で`use-package :ensure t`しただけだと、Eatのterminfoソース(`*.ti`)がデフォルトレシピの`:files`指定から漏れてビルドに失敗する。明示的なレシピ上書きが必要
- Eat側にもvtermの`vterm-keymap-exceptions`に相当する`eat-semi-char-non-bound-keys`があるが、`setq`だけでは反映されず、`eat-update-semi-char-mode-map`と`eat-reload`の呼び出しがセットで必要
- ターミナル内の文字列をコピーするには"emacs"モードに切り替えて普通のEmacsコマンドで選択・コピーする
- vtermは開発が継続している一方、eatは開発がほぼ止まっている。作者の生存とIssue追跡は確認できているが、活発なメンテナンスを期待できる状態ではない

## 参考

- [EmacsのターミナルエミュレーターをvtermからEatに移行しました | Ki_chi@Blog](https://ki-chi.jp/posts/switch-from-vterm-to-emacs-eat/)
- [emacs-libvterm (GitHub)](https://github.com/akermu/emacs-libvterm)
- [emacs-eat (Codeberg)](https://codeberg.org/akib/emacs-eat)
- [Eat User Manual](https://elpa.nongnu.org/nongnu-devel/doc/eat.html)
- [#237 - Is this project abandoned? - akib/emacs-eat - Codeberg.org](https://codeberg.org/akib/emacs-eat/issues/237)
