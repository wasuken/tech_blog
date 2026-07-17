---
title: "go-ts-modeでgofmt(goimports)の設定を修正した"
date: 2026-07-18T03:30:00+09:00
draft: false
ai_assisted: true
tags: ["Emacs", "Go", "tree-sitter", "Linux"]
categories: ["技術"]
description: "go-ts-mode移行後の置き土産にハマった2つの症状（TABキーでインデントが縮む・保存時のgofmtが発火しない）の原因と対処"
---

## TL;DR

- Go開発を`go-ts-mode`(tree-sitterベース)に寄せたら、問題が発生していた。
- 1つ目: `gofmt`(goimportsで代替)を実行した直後は正しく整列されているのに、TABキーを1回押すとインデントが**縮む**。
- 2つ目: `before-save-hook`に`gofmt-before-save`を登録しても、保存時に一切フォーマットされない。しかもエラーは出ない。
- 両方とも「go-mode.el(tree-sitter以前のパッケージ)がgo-ts-modeの存在を想定していない」ことが根っこの原因だった。

## 環境

- Emacs、Go開発は`go-ts-mode`(tree-sitterベース)を使用
- OS: Arch Linux
- `gofmt-command`は`"goimports"`に上書き設定(gofmtの代わりにgoimportsを使う)
- 対象ファイル例: `internal/api/server.go`(構造体フィールドやjsonタグを縦に整列させるコードスタイル)

## 試したこと

### 症状1: 手動でTABキーを押すとインデントが縮む

手動で`M-x gofmt`を実行して整形した直後のファイルは正しく整列されているが、既存行にカーソルを置いてTABキーを押すと、その行のインデントが浅くなる。
思い出したがこれはかなり前から存在してる不備だ。

ちょうどよかったのでここで直してみよう。

切り分けは以下の手順で進めた。

1. `M-x describe-variable RET indent-tabs-mode RET` → 値は`t`。タブを使う設定として正常。
2. 対象行の文字を`C-u C-x =`で確認 → タブ文字であることを確認(`Char: TAB`)。
3. タブの表示幅を疑い、行頭でタブ1個分右に移動してから`C-x =` → `column=8`。`tab-width`も`8`(Emacsのデフォルト)。表示幅そのものは正常。
4. 「短くなった行」と「正常に見える上の行」で`M-m`(back-to-indentation)からの`C-x =`を比較 → 両方`column=16`で一致。インデント幅そのものは揃っていた。

ここで「タブ幅の表示問題ではない」と判明。

真因は次の通り。**`gofmt`は構造体フィールドやjsonタグを縦に整列させるために、構文的に必要な数以上のタブを意図的に挿入する**(整列目的の余分なタブ)。一方`go-ts-mode`はtree-sitterによる構文解析で「その行の構文的必要インデント深さ」だけを計算し、TABキー押下時に行頭の空白をその計算値に置き換える。gofmtが足した「整列用の余分なタブ」は構文的には不要なので、go-ts-modeがそれを認識せず削ってしまう。

これはEmacs設定のバグではなく**仕様通りの動作**だ。go-ts-modeには「整列のための余分な空白」を認識する機能がそもそもない。

調査の途中で、副原因ももう1つ見つかった。`go-ts-mode-indent-offset`が`4`に設定されていた。

```elisp
(use-package treesit
  :straight (:type built-in)
  :config
  (setq treesit-font-lock-level 4)
  (setq go-ts-mode-indent-offset 4))
```

`go-ts-mode-indent-offset = 4`だと「1インデントレベル=4列」で計算される。一方`tab-width`はEmacsデフォルトの`8`のまま。gofmtの出力は「1インデントレベル=タブ1個(実質8列相当)」が前提なので、この不一致でTABキー押下時の再インデント計算が4列刻みでずれる。

対処は`go-ts-mode-indent-offset`を`tab-width`と同じ値に揃えるだけ。

```elisp
;; 変更前
(setq go-ts-mode-indent-offset 4)
```

```elisp
;; 変更後(効果確認済み)
(setq go-ts-mode-indent-offset 8)
```

これで症状は解消した。

#### 色々考えたがタブ表示幅は4のままにしたい

「表示幅は8より4が好み」という要望が自分の中にあった。結論としては、**`tab-width`と`go-ts-mode-indent-offset`の値が一致してさえいれば**8である必要はない。両方4に揃えても問題ない。

懸念点として「`tab-width`を変えるとgofmtの出力(タブ文字の個数)がおかしくならないか」という疑問が浮かんだが、これは誤解だった。`gofmt`はタブの表示幅(`tab-width`)を一切参照しない。gofmtの整列ロジックは常に「1インデントレベル=タブ文字1個」という文字ベースのルールで動作し、Emacs側の表示設定には依存しない。`tab-width`はEmacsの表示専用の変数であり、ファイル内容(タブ文字そのものの個数)には一切影響しない。

最終的には以下に落ち着いた。実際に動作確認済み。

```elisp
(setq-default tab-width 4)
(setq go-ts-mode-indent-offset 4)
```

### 症状2: 保存時の自動フォーマット(before-save-hook)が効かない

症状発覚時のコードはこうなっていた。

```elisp
(setq gofmt-command "goimports")
(add-hook 'go-ts-mode-hook
          (lambda ()
            (add-hook 'before-save-hook #'gofmt-before-save nil t)))
(add-hook 'go-ts-mode-hook #'eglot-ensure)

(use-package go-mode
  :config
  (setq gofmt-command "goimports"))

(add-hook 'go-ts-mode-hook #'eglot-ensure)
(add-hook 'go-ts-mode-hook
          (lambda ()
            (add-hook 'before-save-hook #'gofmt-before-save nil t)))
```

`go-ts-mode-hook`へのeglot-ensure登録とgofmt-before-save登録がそれぞれ2重に書かれていたのは、後から見返すと単純にコピペ跡が残っていただけだった。

手動で`M-x gofmt`を実行すると正しく整形される。しかし`C-x C-s`(保存)だけではフォーマットされない。しかも`*Messages*`バッファにエラーは一切出ない。実行すると即死どころか、何も起きないので気づくのに時間がかかった。

切り分けはこう進めた。

1. `which goimports` → `~/go/bin/goimports`(シェルPATHには存在)。
2. Emacs内で`M-: (executable-find "goimports")` → 同じパスが返る。EmacsのPATH解決も正常。
3. バッファのメジャーモードを確認 → `go-ts-mode`になっていることを確認。
4. `M-x describe-variable RET before-save-hook RET` → ローカル値に`(eglot--signal-textDocument/willSave gofmt-before-save t)`が登録されていることを確認。フック登録自体は正常。
5. 保存前後で`*Messages*`を確認 → エラーなし、フォーマットもされない。
6. `M-x find-function RET gofmt-before-save RET`で関数定義元にジャンプ。中身は以下だった。

```elisp
(when (eq major-mode 'go-mode) (gofmt))
```

原因はこれだった。**`gofmt-before-save`(go-mode.el由来)の実装が`(eq major-mode 'go-mode)`という完全一致判定になっている**。`derived-mode-p`ではなく`eq`なので、`go-ts-mode`で開いている場合は条件を満たさず、`gofmt`(実体はgoimports呼び出し)が一切呼ばれずに素通りする。go-mode.el側がtree-sitter版モードの存在を考慮していない古い実装であることが根本原因だ。

対処として、自前のラッパー関数を定義し、`derived-mode-p`で`go-mode`と`go-ts-mode`の両方を判定させることにした。

```elisp
(defun my/gofmt-before-save ()
  "gofmt-before-save の go-ts-mode 非対応を回避するラッパー."
  (when (derived-mode-p 'go-mode 'go-ts-mode)
    (gofmt)))
```

登録側もこちらに差し替え、重複していたフック登録も1回に整理した。

```elisp
(add-hook 'go-ts-mode-hook #'eglot-ensure)
(add-hook 'go-ts-mode-hook
          (lambda ()
            (add-hook 'before-save-hook #'my/gofmt-before-save nil t)))
```

#### go-modeブロックを消したら`(void-function gofmt)`

`go-ts-mode`に一本化しているつもりで、記事化作業中に`use-package go-mode`ブロック自体を削除した。すると保存時に`Before-save hook error: (void-function gofmt)`というエラーが発生した。

原因は単純で、`gofmt`関数自体は`go-mode.el`パッケージ内で定義されている。`use-package go-mode`を削除するとパッケージがロードされなくなり、`go-mode`は使っていなくても`gofmt`関数だけは存在しなくなっていた。

対処は、`go-mode`パッケージは**ロードだけはする**が、自動起動(モードとしての使用)はさせない設定にすること。

```elisp
(use-package go-mode
  :defer t)
```

`:defer t`によりパッケージはロードされる(=`gofmt`関数は使える)が、`.go`ファイルを開いても`go-mode`が自動起動することはない。実際に使うのは`go-ts-mode`のままだ。

この修正後、保存時に以下のメッセージが出て正常にフォーマットされることを確認した。

```
Calling gofmt: goimports (-srcdir ~/dev/myproject/internal/api/server.go -w /tmp/gofmtjqOqtQ.go)
```

## 結論：go-mode / go-ts-mode 対応マップ

| 項目 | go-mode | go-ts-mode(素の状態) | go-ts-mode(対処後) |
|---|---|---|---|
| gofmt後、TABキーでインデントが崩れない | ✅ | ❌ | ✅(indent-offsetをtab-widthに揃える) |
| `gofmt-before-save`が発火する | ✅ | ❌(`eq`判定でスルーされる) | ✅(`derived-mode-p`ラッパーで対応) |
| `gofmt`関数自体が使える | ✅ | ❌(go-mode未ロードだと`void-function`) | ✅(`:defer t`でロードのみ) |

## 現実的な代替案

1. `go-ts-mode-indent-offset`を`tab-width`と一致させる(値は8でも4でもよい。とにかく揃える)。
2. `gofmt-before-save`をそのまま使わず、`derived-mode-p`で判定する自前ラッパーを`before-save-hook`に登録する。
3. `go-mode`パッケージは`:defer t`でロードだけ行い、モードとしては起動させない(`gofmt`関数を生かすため)。

## 最終的な設定

```elisp
;; Tree-sitter設定内
(setq go-ts-mode-indent-offset 4)  ; tab-widthと一致させること

;; Go
;; gofmt-before-save は (eq major-mode 'go-mode) を直接チェックしており
;; go-ts-mode では発火しないため、derived-mode-p で判定する自前関数を使う。
;; gofmt 関数自体は go-mode.el で定義されているため、go-mode 自動起動はさせず
;; パッケージのロードだけ行う。
(use-package go-mode
  :defer t)

(setq gofmt-command "goimports")  ; gofmtの代わりにgoimportsを使う

(defun my/gofmt-before-save ()
  "gofmt-before-save の go-ts-mode 非対応を回避するラッパー."
  (when (derived-mode-p 'go-mode 'go-ts-mode)
    (gofmt)))

(add-hook 'go-ts-mode-hook #'eglot-ensure)
(add-hook 'go-ts-mode-hook
          (lambda ()
            (add-hook 'before-save-hook #'my/gofmt-before-save nil t)))
```

(別途`tab-width`は`(setq-default tab-width 4)`などで揃えておくこと)

# 所感

動かない時はソースを直接読みに行くのが結局一番早い。`M-x find-function`で`gofmt-before-save`の中身を見るまで、まさか`eq`で完全一致判定しているとは思わなかった。tree-sitter系モードへの移行は便利な反面、周辺の古いパッケージが新モードの存在を想定していないケースが地味に転がっている。今回はたまたま`derived-mode-p`で回避できたが、他の`go-mode`前提のパッケージでも同じ罠がありそうな気配はある。

## 参考

- go-mode.el 公式リポジトリ: [dominikh/go-mode.el](https://github.com/dominikh/go-mode.el)
- go-mode.el README（`gofmt-before-save`等の機能説明）: [dominikh/go-mode.el/blob/master/README.md](https://github.com/dominikh/go-mode.el/blob/master/README.md)
- `gofmt-before-save`をminor mode化する提案PR（未マージ、議論継続中）: [dominikh/go-mode.el/pull/426](https://github.com/dominikh/go-mode.el/pull/426)
- `before-save-hook`関連の過去Issue（内容は今回の症状と完全に一致するわけではなく、`go-remove-unused-imports`絡みの設定ミスに関するもの。フック周りで詰まった過去事例として参考まで）: [dominikh/go-mode.el/issues/106](https://github.com/dominikh/go-mode.el/issues/106)
- Spacemacsでの`gofmt-before-save`利用例（go layer）: [syl20bnr/spacemacs/pull/11082/files](https://github.com/syl20bnr/spacemacs/pull/11082/files)
