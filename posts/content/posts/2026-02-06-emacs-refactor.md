---
title: "Emacsのdotfilesをモジュール化してメンテナンス性を向上させた話"
date: 2026-02-06T23:30:00+09:00
draft: false
tags: ["Emacs", "dotfiles", "リファクタリング", "設定管理"]
categories: ["技術", "開発環境"]
description: "900行の肥大化したEmacs設定をファイル分割し、requireとloadの違いを理解して安定した環境を構築"
---

# Emacsのdotfilesをモジュール化してメンテナンス性を向上させた話

## 背景

約900行に肥大化した`mypackage.el`を整理し、機能ごとにファイル分割してメンテナンス性を向上させるリファクタリングを実施した。

## 課題

- **単一ファイルの肥大化**: `mypackage.el`が900行超えで見通しが悪い
- **機密情報の混在**: API keyがコード内に散在
- **使っていない設定**: コメントアウトされた設定が残存
- **パッケージの把握困難**: 何を使っているか不明瞭

## 新しいディレクトリ構成

```
dotfiles/emacs/
├── init.el                    # エントリーポイント
├── early-init.el              # 起動高速化
├── core/
│   ├── env.el                 # 環境変数・基本設定
│   ├── custom.el              # UI基本設定
│   ├── keymap.el              # キーバインド
│   └── util.el                # ユーティリティ関数
├── packages/
│   ├── manager.el             # straight.el設定
│   ├── core.el                # 基盤パッケージ
│   ├── completion.el          # 補完系 (Vertico, Corfu)
│   ├── search.el              # 検索系 (Consult, Embark)
│   ├── git.el                 # Magit等
│   ├── lsp.el                 # Eglot等
│   ├── languages.el           # 言語別設定
│   ├── ai.el                  # GPTel, Ollama
│   ├── writing.el             # Denote, Org, Markdown
│   ├── ui.el                  # テーマ、アイコン
│   └── optional.el            # たまに使うもの
├── templates/                 # Tempelテンプレート
└── docs/
    └── README.md
```

## 重要な学び: `require` vs `load`

### 問題: `require`でパッケージが読み込まれない

当初、`init.el`で`(require 'completion)`のように読み込んでいたが、以下の問題が発生:

1. **`provide`のキャッシュ**: 一度`require`で読み込むと、`features`リストに記録され、再度`require`してもスキップされる
2. **キーバインドの上書き**: `keymap.el`を先に読み込んでも、後からパッケージが上書き

### 解決策: `load`を使用

```elisp
;; ❌ これだとキャッシュされる
(require 'completion)
(require 'git)

;; ✅ loadは毎回実行される
(load (expand-file-name "packages/completion.el" dotfiles-emacs-dir))
(load (expand-file-name "packages/git.el" dotfiles-emacs-dir))
```

**`load`の利点:**
- `provide`の有無に関係なく確実に実行
- 設定変更後の再読み込みが確実
- キーバインドなど、即座に実行したい設定に最適

### 最終的な`init.el`

```elisp
;;; init.el --- Wasu's Emacs Configuration -*- lexical-binding: t; -*-

;; package.elを無効化
(setq package-enable-at-startup nil)

;; Load path
(defvar dotfiles-emacs-dir (expand-file-name "~/dotfiles/emacs/"))

(add-to-list 'load-path (expand-file-name "core" dotfiles-emacs-dir))
(add-to-list 'load-path (expand-file-name "packages" dotfiles-emacs-dir))

;; Core configuration
(load (expand-file-name "core/env.el" dotfiles-emacs-dir))
(load (expand-file-name "core/custom.el" dotfiles-emacs-dir))

;; Package management
(load (expand-file-name "packages/manager.el" dotfiles-emacs-dir))

;; Custom file (secrets) - パッケージより先に読み込む
(setq custom-file (expand-file-name "config.el" user-emacs-directory))
(when (file-exists-p custom-file)
  (load custom-file))

;; Core packages & Utils
(load (expand-file-name "packages/core.el" dotfiles-emacs-dir))
(load (expand-file-name "core/util.el" dotfiles-emacs-dir))

;; Packages (全てload)
(load (expand-file-name "packages/completion.el" dotfiles-emacs-dir))
(load (expand-file-name "packages/search.el" dotfiles-emacs-dir))
(load (expand-file-name "packages/lsp.el" dotfiles-emacs-dir))
(load (expand-file-name "packages/languages.el" dotfiles-emacs-dir))
(load (expand-file-name "packages/ui.el" dotfiles-emacs-dir))
(load (expand-file-name "packages/git.el" dotfiles-emacs-dir))
(load (expand-file-name "packages/writing.el" dotfiles-emacs-dir))
(load (expand-file-name "packages/ai.el" dotfiles-emacs-dir))
(load (expand-file-name "packages/optional.el" dotfiles-emacs-dir))

;; Font (optional)
(let ((font-config (expand-file-name "core/font.el" dotfiles-emacs-dir)))
  (when (file-exists-p font-config)
    (load font-config)))

;; Keymap (最後に読み込んで上書きを防ぐ)
(load (expand-file-name "core/keymap.el" dotfiles-emacs-dir))

(provide 'init)
;;; init.el ends here
```

## 機密情報の分離

`~/.emacs.d/config.el`に機密情報を集約:

```elisp
;;; config.el --- Private configuration

;; API Keys
(setq gemini-api-key "your-key")
(setq habitica-uid "your-uid")
(setq habitica-token "your-token")

;; Ollama
(setq ollama-host "localhost")
(setq ollama-port 11434)
(setq ollama-model "qwen2.5:7b-instruct")

;; Mastodon
(setq mastodon-instance-url "https://mstdn.jp/")
(setq mastodon-active-user "wasulisp")

(provide 'config)
```

このファイルは.emacs.dに配置するのでバージョン管理外で管理する。

## early-init.elで起動高速化

```elisp
;;; early-init.el --- Early initialization

;; package.elを無効化 (straight.el使用のため)
(setq package-enable-at-startup nil)

;; GC閾値を一時的に上げて起動高速化
(setq gc-cons-threshold most-positive-fixnum)

;; 起動後に閾値を戻す
(add-hook 'emacs-startup-hook
          (lambda ()
            (setq gc-cons-threshold (* 16 1024 1024))))

(provide 'early-init)
```

これにより`~/.emacs.d/elpa`との競合を防ぎ、起動を高速化する。

## パッケージ分割の例

中身について抜粋となる。

詳しくはリポジトリを参照。

[GitHub - wasuken/dotfiles at dev](https://github.com/wasuken/dotfiles/tree/dev)

### completion.el

```elisp
;;; completion.el --- Completion framework

;; Vertico - 縦型補完UI
(use-package vertico
  :config
  (setq vertico-cycle t)
  (vertico-mode +1))

;; Corfu - インライン補完
(use-package corfu
  :demand t
  :config
  (setq corfu-cycle t
        corfu-auto t
        corfu-auto-prefix 1)
  (global-corfu-mode +1))

;; Orderless - 柔軟な検索
(use-package orderless
  :config
  (setq completion-styles '(orderless basic)))

(provide 'completion)
```

### git.el

```elisp
;;; git.el --- Git integration

(use-package magit
  :bind ("C-x g" . magit))

(use-package diff-hl
  :hook ((magit-pre-refresh . diff-hl-magit-pre-refresh)
         (magit-post-refresh . diff-hl-magit-post-refresh))
  :config
  (global-diff-hl-mode +1))

(provide 'git)
```

## セットアップ手順

```bash
# シンボリックリンク作成
ln -sf ~/dotfiles/emacs/init.el ~/.emacs.d/init.el
ln -sf ~/dotfiles/emacs/early-init.el ~/.emacs.d/early-init.el

# 機密情報ファイル作成
touch ~/.emacs.d/config.el
# (API key等を記述)

# Emacs起動
emacs
```

## トラブルシューティング

### キーバインドが効かない

**原因**: パッケージが後から上書き

**解決**: `keymap.el`を`init.el`の最後で`load`

### パッケージが見つからない

**原因**: `require`のキャッシュ

**解決**: `load`を使用するか、完全再起動

```bash
pkill emacs
emacs
```

### 環境変数が未定義

**原因**: `config.el`の読み込み順序

**解決**: `config.el`をパッケージ読み込み前に`load`

## 成果

- ✅ **見通し向上**: 機能ごとにファイル分割
- ✅ **保守性向上**: 変更箇所が明確
- ✅ **セキュリティ**: 機密情報を分離
- ✅ **動作安定**: `load`による確実な読み込み

## まとめ

Emacsの設定ファイルをモジュール化する際は、`require`と`load`の違いを理解することが重要。
特にキーバインドや即座に実行したい設定は`load`を使用し、読み込み順序を制御することで、安定した環境を構築する。
