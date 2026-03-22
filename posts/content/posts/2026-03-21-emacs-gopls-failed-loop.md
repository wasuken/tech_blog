---
title: "EmacsでgoplsがSIGKILLされて補完できない問題"
date: 2026-03-22T00:00:00+09:00
draft: false
tags: ["emacs", "go", "gopls", "eglot", "lsp"]
categories: ["開発環境"]
---

## 症状

Emacsで `.go` ファイルを開くと以下のログが無限ループする。
```
[eglot] Asking EGLOT (mydns/(go-ts-mode go-mod-ts-mode)) politely to terminate
[jsonrpc] Server exited with status 2
[eglot] Reconnected!
[eglot] Connected! Server 'gopls' now managing '(go-ts-mode go-mod-ts-mode)' buffers in project 'mydns'.
[jsonrpc] (warning) Sentinel for EGLOT (...) still hasn't run, deleting it!
[jsonrpc] Server exited with status 9
[eglot] Reconnected! [2 times]
Error running timer: (error "Selecting deleted buffer")
```

`status 9` は SIGKILL。gopls が起動→即死→reconnect を繰り返し、補完が一切効かない状態。

`fmt.` と打っても候補が出ない、もしくは関係のないゴミ候補が出る。手動で `M-x completion-at-point` を叩いても `No match`。

## 調査

### eglot-events-bufferで何も見えない

まず `M-x eglot-events-buffer` でgoplsとのやり取りを確認しようとしたが、何も表示されなかった。

設定を確認すると原因がわかった。
```elisp
(setq eglot-events-buffer-config '(:size 0 :format short))
```

`:size 0` でイベントバッファを無効化していた。デバッグのためにまず `:size 100` 以上に変更する必要がある。

### completion-at-pointでNo match

`M-x completion-at-point` を手動で叩いても `No match`。goplsまでリクエストが届いていないことが確定した。

### eglot-reconnectでループ開始

`M-x eglot-reconnect` を試したところ、上記のループが始まった。

### goplsバイナリ自体は正常

goplsが壊れている可能性を疑った。
```bash
which gopls
# /home/wasu/go/bin/gopls

gopls version
# golang.org/x/tools/gopls v0.21.1
```

バイナリは正常にインストールされていた。

### デーモンモードの調査

goplsのログに以下が出ていた。
```
serve.go:173: Gopls LSP daemon: listening on tcp network, address :12345...
```

デーモンモードで動いているgoplsとeglotが競合している可能性を疑った。`ss -atn | grep 12345` で確認したが該当なし。デーモンは関係なかった。

### pkill -9 goplsは効かなかった

プロセスが残骸として残っている可能性を疑い `pkill -9 gopls` を試したが、ループは継続した。

### 設定ファイルの確認

lsp.elを確認したところ、`eglot-managed-mode-hook` に刺さっているパッケージが複数あった。

- `eglot-x`
- `eglot-tempel`
- `eldoc-box`
- `eglot-signature-eldoc-talkative`
- `flymake-collection`
- `cape`

まず `eglot-x` を疑った。eglotの内部フックを書き換えるパッケージで、壊れると症状がわかりにくい。`:disabled t` にして `package-delete` で削除したが、ループは継続した。

### 素のeglotに削る

lsp.elをeglotだけの最小構成に削った。
```elisp
(use-package eglot
  :config
  (setq eglot-events-buffer-config '(:size 100 :format full)
        eglot-send-changes-idle-time 1.0)
  (add-to-list 'eglot-server-programs
               '((go-ts-mode go-mod-ts-mode) . ("gopls" "-remote=auto"))))
```

**補完が動いた。** eglot周辺のパッケージが原因と確定。

### 一個ずつ戻す

以下の順で追加して都度 `fmt.` で補完を確認した。

1. `flymake` → ok
2. `eglot-tempel` → ok
3. `eldoc-box` → ok
4. `eglot-signature-eldoc-talkative` → ok
5. `consult-eglot` → ok
6. `jsonrpc` → ok
7. `flymake-collection` → ok
8. `cape` → ok
9. `eglot-x` → ok

**全部戻しても動いた。**

## 結果

二分探索では犯人を特定できなかった。

ただし切り分けの過程で `eglot-x` を一度 `package-delete` で削除し、`:vc` で再取得していた。
```elisp
(use-package eglot-x
  :straight nil
  :vc ( :fetcher github :repo "nemethf/eglot-x")
  :after eglot
  :config
  (eglot-x-setup))
```

straight.elや `:vc` はキャッシュを使うため、明示的に削除して再取得しないと古いバージョンのままになることがある。**`eglot-x` の古いバージョンが原因だった可能性が高いが、断定はできていない。**

## まとめ

| やったこと | 結果 |
|---|---|
| `eglot-events-buffer` で確認 | `:size 0` で無効化されていて何も見えなかった |
| `completion-at-point` | `No match`、goplsまで届いていない |
| goplsバイナリ確認 | 正常 |
| デーモンモード調査 | 関係なかった |
| `pkill -9 gopls` | 効かなかった |
| 素のeglotに削る | 補完が動いた |
| パッケージを一個ずつ戻す | 全部okだった |
| `eglot-x` を再取得済み | 以降ループが止まった（推定原因） |

eglotのフックに刺さるパッケージが壊れると症状がわかりにくい。補完が死んだらまず素のeglotに戻して二分探索するのが有効だった。

こんなことで数時間溶かすことになるとは。
