---
title: "Janet言語をEmacsで書くまでにハマった話（treesit-auto・jpm・spork）"
date: 2026-09-06T21:00:00+09:00
draft: true
tags: ["Janet", "Emacs", "Arch Linux", "jpm"]
categories: ["技術"]
description: "janetというLisp方言の環境構築を行ったログ"
ai_assisted: true
---

備忘録なので本文気合いれるつもりもなかったのでチェックだけして

実際に行った手順ややり取りを元にAIで文章を生成した。

## この記事でわかること

- Emacs で Janet を書ける状態にするまでの一連の詰まりポイント
- `treesit-auto` の janet レシピが `:lang` を間違えていた話（v1.0.5以前が該当、v1.0.8で修正済み）
- eglot を straight.el で git 管理下に置くと Emacs 本体の未pushコミットとして誤検出される話
- Arch で jpm が `janet` パッケージに含まれない理由と、jpm を git 版に切り替える手順
- jpm と janet のインストール元を分けたときに起きるヘッダパスのズレの直し方
- Arch のネイティブパッケージ（spork）が本体アップデートに追いついていない時の見分け方と回避策

要するに「Janet 自体の話」ではなく「Arch + Emacs で最新のツールチェーンを組もうとすると起きる、環境構築あるあるの詰め合わせ」です。

手っ取り早く動く状態だけ知りたい人は [→ 最短手順](#最短手順詰まらずにやる場合) へ。

## 動機

新しい言語として [Janet](https://janet-lang.org/) を触ることにした。関数型・手続き型の両方をサポートする軽量な Lisp 系言語で、コア一式（インタプリタ・コンパイラ・アセンブラ・PEG）が1MB未満に収まる。組み込み用途やシステムスクリプティングを想定していて、ファイバ（グリーンスレッド）と PEG が標準で入っているのが特徴。

Emacs で書く前提で環境を組んだが、想定以上に詰まったのでログとして残す。結論だけ知りたい人は最後の「まとめ」だけ読めばいい。

## 1. tree-sitter の major mode を入れ忘れる

Emacs 側の設定はもともとこうなっていた。

```elisp
(setq treesit-language-source-alist
      '(...
        (janet-simple . ("https://github.com/sogaiu/tree-sitter-janet-simple"))
        ...))
```

これは tree-sitter の **grammar**（構文木を作るための C ライブラリ）の取得先でしかなく、`.janet` を開いたときに使う **major mode** とは別物だった。実際に major mode を提供するパッケージ [janet-ts-mode](https://github.com/sogaiu/janet-ts-mode) を入れていなかったので、`.janet` を開いても最後まで `fundamental-mode` のままだった。

エディタ支援の全体像は [janet-editor-and-tooling-info の emacs.md](https://github.com/sogaiu/janet-editor-and-tooling-info/blob/master/doc/emacs.md) にまとまっている。Janet 用の major mode は少なくとも2つある。

> There are at least two major modes providing Janet support: janet-mode, janet-ts-mode

REPL 側は stdio 方式と netrepl 方式の2系統があり、今回は同じ作者 (sogaiu) のものという理由で [ajrepl](https://github.com/sogaiu/ajrepl) を採用した。straight.el での導入はこう。

```elisp
(straight-use-package
 '(janet-ts-mode :host github :repo "sogaiu/janet-ts-mode" :files ("*.el")))

(use-package janet-ts-mode
  :straight t
  :mode "\\.janet\\'")

(straight-use-package
 '(ajrepl :host github :repo "sogaiu/ajrepl" :files ("*.el" "ajrepl")))

(use-package ajrepl
  :straight t
  :after janet-ts-mode
  :config
  (add-hook 'janet-ts-mode-hook #'ajrepl-interaction-mode))
```

## 2. 毎回勝手に clone される grammar の正体

major mode を足した後、こういう警告が出るようになった。

```
Warning (treesit): The installed language grammar for janet cannot be located
or has problems (symbol-error):
/home/user/.emacs.d/tree-sitter/libtree-sitter-janet.so:
undefined symbol: tree_sitter_janet
```

ファイルを開くたびに勝手に git clone が走る、というオマケ付きだった。

原因は `treesit-auto` パッケージが内蔵している言語レシピだった。手元でリポジトリを clone してヒストリを追ったところ、2023年に追加された時点のレシピはこうなっていた。

```elisp
,(make-treesit-auto-recipe
  :lang 'janet
  :ts-mode 'janet-ts-mode
  :remap 'janet-mode
  :url "https://github.com/sogaiu/tree-sitter-janet-simple"
  :ext "\\.janet\\'")
```

`:lang` は `janet` なのに URL の中身は `tree-sitter-janet-simple`。ビルドされる共有ライブラリが吐く symbol は `tree_sitter_janet_simple` なので、`janet` という名前で期待される `tree_sitter_janet` と一致しない。これが symbol-error の正体で、`:lang` が壊れているせいで「入っていない」と毎回判定され、ファイルを開くたびに再取得が走っていた。

このバグは 2024-09-22 の commit [`74587ac`](https://github.com/renzmann/treesit-auto)「delete latex, fix janet-simple, and fix markdown」で `:lang 'janet-simple` に修正済み。`v1.0.5` 以前が該当し、`v1.0.8` 以降は直っている。

対処として、まず壊れた `.so` を削除し、`treesit-auto` を最新化するか、レシピから janet を除外した。

```elisp
(use-package treesit-auto
  :config
  (setq treesit-auto-recipe-list
        (seq-remove (lambda (r) (eq (treesit-auto-recipe-lang r) 'janet))
                    treesit-auto-recipe-list))
  (treesit-auto-add-to-auto-mode-alist 'all)
  (setq treesit-auto-install t)
  (global-treesit-auto-mode +1))
```

janet は `treesit-language-source-alist` と `janet-ts-mode` の `:mode` だけで完結しているので、`treesit-auto` に任せる必要は元からなかった。

## 3. eglot が「279コミット未pushで止まる」

Emacs パッケージ全体を最新化しようとしたところ、`straight-pull-all` の途中でこう出た。

```
In repository eglot, branch master has 279 commits unpushed to origin/master
```

`eglot` は Emacs 本体にバンドルされている GNU ELPA の :core package で、GitHub 上の `joaotavora/eglot` リポジトリは Emacs 本体の開発と同期している。

> This is a GNU ELPA :core package. Avoid adding functionality that is not available in the version of Emacs recorded above or any of the package dependencies.

つまり自分の設定ミスではなく、Emacs 本体のリポジトリを straight.el 経由で git clone してしまっていたのが原因。built-in 指定に切り替えて解決。

```elisp
(use-package eglot
  :straight (:type built-in))
```

同じ構造の罠は `project` `flymake` `xref` `jsonrpc` あたりにも当てはまる。

## 4. jpm がパッケージに含まれない

Arch では `janet` パッケージの依存関係を見ると `jpm` は optional depends 扱いで、`pacman -S janet` だけでは付いてこない。

出典: [Arch Linux - janet 1.41.2-3](https://archlinux.org/packages/extra/x86_64/janet/)

```sh
sudo pacman -S jpm
```

が必要だった。

## 5. jpm はバージョンを切らない

janet-lsp（後述）を試すために jpm を最新化しようとしたが、[janet-lang/jpm](https://github.com/janet-lang/jpm) には release タグが無く、常に master を追う運用だった。janet 本体の README にも案内がある。

> To install [Spork](#) or [JPM](#) from source, use `make install-jpm-git` or `make install-spork-git`.

Makefile の中身を見ると、この target は一時的にソースからビルドした `build/janet` を使って jpm の `bootstrap.janet` を実行するだけで、システムの `janet` コマンドには一切触れない。

```makefile
install-jpm-git: $(JANET_TARGET)
	git clone --depth=1 --branch='$(JPM_TAG)' https://github.com/janet-lang/jpm.git build/jpm
	cd build/jpm && ... $(RUN) ../../$(JANET_TARGET) ./bootstrap.janet
```

なので pacman 管理の `janet` 本体はそのままに、`jpm` だけ git 版へ切り替えるという構成が成立する。インストール先が `/usr/local` 配下になるため、実行には sudo が必要。

```sh
sudo pacman -R jpm
cd ~/src && git clone https://github.com/janet-lang/janet.git
cd janet && sudo make install-jpm-git
```

## 6. jpm と janet のヘッダパスがズレる

jpm を `/usr/local/bin` に入れた結果、C 拡張のビルド時にこう怒られた。

```
error: unabled to find janet.h: checked /usr/local/include/janet/janet.h, ...
```

jpm はヘッダの場所を「自分がインストールされた場所基準」で決め打ちする仕様。

> **JANET_HEADERPATH** ... If not provided, will default to `<jpm script location>/../include/janet`

出典: [jpm man page (gist)](https://gist.github.com/sogaiu/fd13231e0f239e3b88a8019780f064f4)

jpm を `/usr/local` に、janet 本体を pacman 経由で `/usr` に、と別々のところに置いた時点でこのズレは仕様通り発生する。janet 本体まで手動ビルドに寄せて揃える手もあったが、それだとせっかく pacman に任せていたアップデート管理を手放すことになるので、jpm 側のデフォルト設定だけ書き換えて解決した。

```sh
sudo sed -i \
  's|:headerpath "/usr/local/include/janet"|:headerpath "/usr/include/janet"|' \
  /usr/local/lib/janet/jpm/default-config.janet
```

## 7. spork の ABI 不一致

janet 本体を 1.42.0 に上げた直後、`spork/json` の import でこう出た。

```
error: could not load native /usr/lib/janet/spork/json.so:
config mismatch - host 1.42.0(0001) vs. module 1.41.2(0001)
```

Arch の janet パッケージが 2026-09-01 に 1.42.0-1 へ更新された一方、janet-spork パッケージのビルドは 2026-07-11 で止まっていた。

- janet 1.42.0-1 への更新: [arch-commits ML](http://www.mail-archive.com/arch-commits@lists.archlinux.org/msg1159176.html)
- janet-spork 1.2.0-2 のビルド日: [Arch Linux - janet-spork](https://archlinux.org/packages/extra/x86_64/janet-spork/)

`pacman -Syu` を打っても新しい spork ビルドがまだリポジトリに来ていなければ直らない。純 Janet のモジュール（`spork/argparse` など `.so` を持たないもの）は問題なく使えるので、ネイティブ拡張入りのモジュールだけ待つか、`jpm -l install spork` でローカルに自前ビルドするしかない。

結局この時点で、pacman 管理の spork は本体更新への追従が遅れて詰むことが分かったので、最初から spork も jpm 経由のソースビルドに寄せるのが安全という結論に落ち着いた。

## 最短手順（詰まらずにやる場合）

紆余曲折を全部飛ばして最初からこの順でやれば、上のハマりポイントには当たらない。pacman 管理のパッケージ（jpm・spork）はビルドが本体更新に追いつかず壊れやすいので、janet 本体以外はソースビルドに寄せる。

### 1. janet本体は pacman、jpm はソースビルド

```sh
sudo pacman -S janet
sudo pacman -R jpm 2>/dev/null  # optional depsで入ってたら消す
cd ~/src && git clone https://github.com/janet-lang/janet.git
cd janet && sudo make install-jpm-git
```

jpm に release タグは無く常に master を追う運用なので、pacman 版を待つ意味がない。

### 2. jpmのヘッダパスを直す

```sh
sudo sed -i \
  's|:headerpath "/usr/local/include/janet"|:headerpath "/usr/include/janet"|' \
  /usr/local/lib/janet/jpm/default-config.janet
```

janet本体をpacman管理のまま使うなら、この1行は必ず要る。

### 3. spork もソースビルド

```sh
cd ~/src/janet && sudo make install-spork-git
```

pacman 版の `janet-spork` は本体アップデートにビルドが追いつかず `config mismatch` で壊れることがある。最初からソースビルドにしておけば、この不一致に当たらない。

### 4. Emacs設定は一式まとめて

```elisp
;; tree-sitter grammar
(setq treesit-language-source-alist
      '((janet-simple . ("https://github.com/sogaiu/tree-sitter-janet-simple"))))

;; major mode
(straight-use-package
 '(janet-ts-mode :host github :repo "sogaiu/janet-ts-mode" :files ("*.el")))
(use-package janet-ts-mode
  :straight t
  :mode "\\.janet\\'")

;; REPL
(straight-use-package
 '(ajrepl :host github :repo "sogaiu/ajrepl" :files ("*.el" "ajrepl")))
(use-package ajrepl
  :straight t
  :after janet-ts-mode
  :config
  (add-hook 'janet-ts-mode-hook #'ajrepl-interaction-mode))

;; eglotはbuilt-in指定（git管理下に置かない）
(use-package eglot
  :straight (:type built-in))

;; treesit-autoを使うならjanetは除外（v1.0.8未満は特に注意）
(use-package treesit-auto
  :config
  (setq treesit-auto-recipe-list
        (seq-remove (lambda (r) (eq (treesit-auto-recipe-lang r) 'janet))
                    treesit-auto-recipe-list))
  (treesit-auto-add-to-auto-mode-alist 'all)
  (setq treesit-auto-install t)
  (global-treesit-auto-mode +1))
```

これに [janet-lsp](https://github.com/CFiggers/janet-lsp) を jpm でインストールしてeglotに繋げば補完まで揃う。

## まとめ

一つ一つは小さいが、全部つながっていた。

| 症状 | 原因 |
|---|---|
| ハイライトはつくが補完もモード名も出ない | major mode パッケージ自体が入っていなかった |
| ファイルを開くたび勝手に clone | `treesit-auto` の janet レシピが `:lang` を間違えていた（v1.0.5以前） |
| straight-pull で謎の未push警告 | eglot が built-in なのに git 管理下に置いていた |
| jpm が無い | Arch では optional depends |
| jpm を最新にしたい | jpm に release タグは無く、常に master |
| C拡張のビルドでヘッダが見つからない | jpm と janet のインストール先を分けたことによる既定パスのズレ |
| spork の import が壊れる | Arch のパッケージリビルドが本体の更新に追いついていない |

最終的な構成は次の形に落ち着いた。

- janet 本体: pacman 管理（アップデート追従は Arch に任せる）
- jpm: `janet-lang/janet` の `make install-jpm-git` で最新化
- spork: 同じく `make install-spork-git` でソースビルド（pacman 版はビルドの追従遅れで壊れやすい）
- jpm のヘッダパスだけ pacman の janet に合わせて手動修正
- Emacs: `janet-ts-mode` + `ajrepl` + `eglot` + [CFiggers/janet-lsp](https://github.com/CFiggers/janet-lsp)

補完まで含めて動くようになったので、次からはようやく言語そのものと向き合える。
