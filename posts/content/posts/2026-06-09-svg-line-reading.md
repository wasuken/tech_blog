---
title: "svg-line: EmacsのステータスバーをSVGで統一する試み"
date: 2026-06-09T00:06:00+09:00
draft: false
ai_assisted: true
tags: ["emacs", "dotfiles", "ui"]
categories: ["tech"]
description: "svg-lineを試した記録。リッチにしようとしたけど結局シンプルに戻した話。"
---

## 元記事

[svg-line: Better Status Bars for Emacs](https://www.chiply.dev/post-svg-line)

Emacsのmode-line、header-line、tab-bar、tab-lineはそれぞれネイティブAPIレベルで挙動が違い、多段表示・右寄せ・アイコン・マウスイベントの扱いがバラバラという問題がある。svg-lineはこれらをSVG画像として描画することで挙動を統一するパッケージ。

SVGにすることで座標ベースのマウスイベント検出ができるのも地味に大きい。ネイティブAPIだとクリックやホバーが*-lineごとに対応がバラバラだったのが一気に解決される。

この記事に触発されて私も少し触ってみようと思った。

あわよくばこのまま置き換えてもいいかとも思う。

## やったこと

### 既存パッケージの整理

svg-lineを試すために一旦以下を削除・調整した。

- `nyan-mode` → 削除
- `minions` → 削除
- `spacious-padding` → `:mode-line-width`の行だけ削除（他の余白設定は残す）

### svg-lineの導入

```elisp
(use-package svg-line
  :straight (:host github :repo "chiply/svg-line"))
```

### mode-lineを書いてみた

最小構成から始めて、バッファ名・git branch・メジャーモード・マイナーモード・行列数の2行構成を目指した。

```elisp
(svg-line-define 'my-mode-line
  :target 'mode-line
  :active #'mode-line-window-selected-p
  :background (lambda () (face-background 'mode-line nil t))
  :foreground (lambda () (face-foreground 'mode-line nil t))
  :content
  (lambda ()
    (list
     (list :left  (list (if (buffer-modified-p) "● " "  ")
                        (buffer-name))
           :right (list (or (and (fboundp 'vc-git--symbolic-ref)
                                 (buffer-file-name)
                                 (vc-git--symbolic-ref (buffer-file-name)))
                            "")))
     (cons (list (symbol-name major-mode))
           (list (format-mode-line "%l:%c"))))))

(svg-line-activate 'my-mode-line)
```

シンプルにはなったが、やはりnyan-modeがないとさみしい。

とはいえ、それ以外はその程度で案外乗り換えてもいいのかもしれない。

## 結局

そもそも現状のままで問題ないと思っていたので、既存モードが利用できなくなっただけだった。

ある程度遊んでからお片付けして終わり。

遊んだだけで終わりました。

今の私には自作してやるぞという覚悟というか熱意が足りなかった気がする。
