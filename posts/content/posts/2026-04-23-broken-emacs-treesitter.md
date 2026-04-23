---
title: "Emacs 30.2 で *-ts-mode が壊れる問題と対処"
date: 2026-04-23T00:00:00+09:00
tags: ["emacs", "tree-sitter", "arch-linux"]
categories: ["emacs"]
---

## 結論

[Mike Olson - Fixing typescript-ts-mode in Emacs 30.2](https://mwolson.org/blog/2026-04-20-fixing-typescript-ts-mode-in-emacs-30-2/)

上記のブログで解説と対応スクリプトが公開されてる。

---

Emacs 30.2 + libtree-sitter 0.26 の組み合わせで `go-ts-mode` や `typescript-ts-mode` などの `*-ts-mode` が動作しない場合、現時点（2026年4月）では以下のワークアラウンドで動作させることができる。

**現時点ではこれが最適解と思う。おそらくEmacs 31 安定版がリリースされるまでこのスクリプトを使い続けることになる。**

```bash
mkdir -p ~/.emacs.d/init
curl -o ~/.emacs.d/init/treesit-predicate-rewrite.el \
  https://raw.githubusercontent.com/mwolson/emacs-shared/master/init/treesit-predicate-rewrite.el
```

`init.el` の早い段階（他の `*-ts-mode` の設定より前）に追加する。

```elisp
(load "~/.emacs.d/init/treesit-predicate-rewrite" nil nil nil t)
```

## 経緯

`go-ts-mode` を開いたとき `*Messages*` バッファに `treesit-query-error` が出てシンタックスハイライトが死んだ。

最初はグラマーの `.so` ファイルが原因だと思い、`treesit-install-language-grammar` で入れ直したり、`~/.emacs.d/tree-sitter/` 以下のファイルを削除して再インストールしたりと試行錯誤した。しかし何をやっても症状が変わらず、グラマー側の問題ではないと判断した。

調べたところ **[Emacs bug#79687](https://debbugs.gnu.org/cgi/bugreport.cgi?bug=79687)** に行き着いた。グラマーの問題ではなく、Emacs 30.2 と libtree-sitter 0.26 の組み合わせ自体が壊れていた。

## 症状

Emacs 30.2 で Go や TypeScript などのファイルを開くと、`*Messages*` バッファに以下のようなエラーが表示され、シンタックスハイライトが一切効かなくなる。

```
Error during redisplay: (jit-lock-function 35) signaled
(treesit-query-error "Syntax error at" 73
"(call_expression function: ((identifier) @font-lock-builtin-face
(#match \"...\" @font-lock-builtin-face)))"
"Debug the query with `treesit-query-validate'")
```

`go-ts-mode` だけでなく `typescript-ts-mode`、`python-ts-mode`、`rust-ts-mode` など、tree-sitter ベースの全モードで同様の問題が発生する。

## 原因

**[Emacs bug#79687](https://debbugs.gnu.org/cgi/bugreport.cgi?bug=79687)** を読んだところ、tree-sitter 側の仕様変更が発端だった。

tree-sitter のクエリには `#match` や `#equal` のような述語を使うことができる。ある時点から tree-sitter はこれらの述語の末尾に `?` または `!` がない場合に構文エラーを返す仕様に変更した。

Emacs はもともと意図的に `#match`（末尾 `?` なし）を使っていた。他のエディタ（Neovim 等）との互換性より「Emacs らしい慣用表現」を優先した設計判断だった。それが tree-sitter 側の変更で突然壊れた形になる。

```
tree-sitter 0.26  →  述語に ? を強制
Emacs 30.2        →  #match のまま → 拒否される ❌
Emacs 31 (master) →  #match? に対応済み ✅
```

Emacs の `master` ブランチでは 2025年11月に対応が完了しているが（bug#79687 クローズ）、`emacs-30` ブランチへのバックポートは 30.2 時点では行われていない。

文字列レベルで `#match` → `#match?` に書き換えることもできない。Emacs 30.2 側の述語ディスパッチャが `match?` を拒否するため、tree-sitter を満足させると Emacs が壊れ、Emacs を満足させると tree-sitter が壊れるというジレンマがある。

## 対処

現時点（2026年4月）でシンプルにバージョンを上げるといった単純な解決策は無さそう。Emacs 31 の master ブランチには修正が入っているが、31 系の安定版リリースはまだ行われておらず、30 系へのバックポートも確認できていない。現行の安定版である Emacs 30.2 を使い続ける限り、以下のワークアラウンドが現状の最適解だ。

### treesit-predicate-rewrite.el

Mike Olson が公開している `treesit-predicate-rewrite.el` を使う。

このファイルは述語をクエリから丸ごと除去し、その述語ロジックを capture-name-as-function fontifier に移すことで、tree-sitter と Emacs 30.2 の両方を満足させる。`go-ts-mode`、`typescript-ts-mode`、`python-ts-mode`、`rust-ts-mode` など主要なモードで動作確認済みだ。

```bash
mkdir -p ~/.emacs.d/init
curl -o ~/.emacs.d/init/treesit-predicate-rewrite.el \
  https://raw.githubusercontent.com/mwolson/emacs-shared/master/init/treesit-predicate-rewrite.el
```

`init.el` に追加する。**他の `*-ts-mode` の設定より前に読み込む**こと。

```elisp
(load "~/.emacs.d/init/treesit-predicate-rewrite" nil nil nil t)
```

`use-package` で `treesit` を管理している場合は `:init` に書くと確実だ。

```elisp
(use-package treesit
  :straight (:type built-in)
  :init
  (load "~/.emacs.d/init/treesit-predicate-rewrite" nil nil nil t)
  :config
  (setq treesit-font-lock-level 3))
```

### やっても意味がなかったこと

グラマーの `.so` ファイルを疑って以下を試したが、どれも効果がなかった。

- `treesit-install-language-grammar` で再インストール
- `~/.emacs.d/tree-sitter/` 以下を削除して入れ直し
- `tree-sitter` パッケージ自体の再インストール

症状はグラマー側ではなく Emacs と libtree-sitter の互換性の問題なので、グラマーをいじっても解決しない。

並行してググっていたが冒頭のサイトが全然見つからず、生成AIも具体的な解決策を出せなかった。最終的に[Planet Emacslife](https://planet.emacslife.com/)でそれらしき内容を見つけて対応できた。

Emacs 関連でハマったら[Planet Emacslife](https://planet.emacslife.com/)を見るのがよさそうだ。

Planet Emacslife 運営者と対策スクリプトを書いて公開してくださった方々へ感謝。

## スクリプトの除去タイミング

30 系へのバックポートが行われるか、Emacs 31 安定版がリリースされ bug#79687 の修正が含まれていることを確認したタイミングで `treesit-predicate-rewrite.el` の読み込みを削除する。

`treesit-predicate-rewrite.el` は述語を含まないクエリには何もしないため、修正済みの Emacs でロードしても副作用はない。削除を忘れても即壊れるわけではないが、不要なコードは消しておくのが無難。

## 参考

- [Emacs bug#79687](https://debbugs.gnu.org/cgi/bugreport.cgi?bug=79687)
- [Fixing typescript-ts-mode in Emacs 30.2 - Mike Olson](https://mwolson.org/blog/emacs/2026-04-20-fixing-typescript-ts-mode-in-emacs-30-2/)
- [treesit-predicate-rewrite.el](https://github.com/mwolson/emacs-shared/blob/master/init/treesit-predicate-rewrite.el)
- [Arch Linux GitLab issue #13](https://gitlab.archlinux.org/archlinux/packaging/packages/emacs/-/work_items/13)
