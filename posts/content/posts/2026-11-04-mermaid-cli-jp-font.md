---
title: "mermaid-cliでSVGの日本語が消えたときの対応"
date: 2026-11-04T15:00:00+09:00
draft: false
ai_assisted: true
tags: ["Linux", "Arch Linux", "Mermaid", "Node.js", "個人開発"]
categories: ["技術"]
description: "mmdcでMarkdown内のMermaidを一括画像化したら日本語が透明になったので、フォントと設定とエイリアスで直した"
---

## TL;DR

- `mmdc` でMarkdown内のMermaidを一括でSVGにしたら、日本語だけ消えた
- 原因は「Linux側の日本語フォント不足」と「HTMLラベル描画（foreignObject）まわり」の2つ
- `noto-fonts-cjk` を入れて、`htmlLabels: false` の設定を渡したら表示された
- 毎回オプションを付けるのがだるいので、エイリアスに押し込んだ

生成AIと一緒に原因を切り分けながら進めた。実際にコマンドを叩いて、表示されるところまで自分の環境で確認している。

## 環境

- ArchLinux
- mermaid-cli（`mmdc`）
- Puppeteer経由でヘッドレスブラウザを使う構成

## やりたかったこと

Markdownに書いたMermaid図をまとめて画像化して差し替えたかった。

```bash
# input.md 内の Mermaid ブロックを画像化して、参照に置き換えた output.md を出す
mmdc -i input.md -o output.md
```

これを実現するまでに２つの問題に遭遇したので対処を説明する

## ①ブラウザが見つからない

最初に、Puppeteer側のブラウザが無いと怒られた。

`mmdc` は内部でヘッドレスブラウザを使って描画するので、そのブラウザが無いと動かない。次のコマンドで headless-shell を入れたら進むらしい。

```bash
# Puppeteer用のブラウザを入れる
npx puppeteer browsers install
```

ただ、私の場合はgoogle-chrome-stableをいれていたのでそのまま指定して行けた。

## ②日本語だけ消える

コンパイル自体は成功する。ただ、出力されたSVGを開くと**日本語の部分だけ透明になったように消えている**。図の枠や矢印は出ているのに、ラベルが空っぽ。

エラーも出ないので、地味に気づきにくい。

### 原因は2つあった

| 原因 | 内容 | 対策 |
|---|---|---|
| 日本語フォント不足 | Linux側にCJKフォントが無く、描画できる字形が無い | `noto-fonts-cjk` を入れる |
| HTMLラベルの描画 | Mermaidはラベルを `foreignObject` 経由のHTMLで描くことがあり、そこで文字が消えた | `htmlLabels: false` を設定で渡す |

フォントだけ入れても直らなかったので、両方やって初めて表示された。

```bash
# 日本語フォント（Arch）
sudo pacman -S noto-fonts-cjk
```

設定ファイルはこんな感じ。

```json
{"htmlLabels": false}
```

```bash
# 設定を -c で渡して実行
mmdc -c config.json -i input.md -o output.md
```

これで日本語が出るようになった。

## めんどくさいのでエイリアス化

毎回 `-c config.json` を用意して付けるのが面倒になった。さらにブラウザのパスも指定したかったので、`.bashrc` / `.zshrc` にエイリアスを1行入れた。

```bash
alias mmdc='PUPPETEER_EXECUTABLE_PATH=/usr/bin/google-chrome-stable mmdc -c <(echo "{\"htmlLabels\": false}")'
```

やっていることは3つ。

- `PUPPETEER_EXECUTABLE_PATH` で、使うChromeを明示する
- `-c <(echo ...)` のプロセス置換で、設定ファイルを作らずにその場で渡す
- 以後は `mmdc -i input.md -o output.md` だけでバグ回避込みで動く

### 注意点

- プロセス置換（`<(...)`）は bash / zsh の機能。素の `sh` では動かない
- エイリアスが `-c` を固定で付けるので、別の設定ファイルを `-c` で渡したいときは衝突しうる。そのときは `\mmdc` でエイリアスを回避して素のコマンドを呼ぶ

## まとめ

| やったこと | 目的 |
|---|---|
| `npx puppeteer browsers install` | ブラウザ未導入エラーの解消 |
| `noto-fonts-cjk` を導入 | 日本語フォント不足の解消 |
| `htmlLabels: false` を `-c` で渡す | ラベル描画で文字が消えるのを回避 |
| エイリアス化 | 毎回のオプション指定をやめる |

**日本語が消えたときは、フォントと描画方式の両方を疑う。片方だけ直しても表示されないことがある。**

## 参考

- [mermaid-js/mermaid-cli (GitHub)](https://github.com/mermaid-js/mermaid-cli)
- [Puppeteer 公式ドキュメント](https://pptr.dev/)
- [Fonts - ArchWiki](https://wiki.archlinux.org/title/Fonts)
