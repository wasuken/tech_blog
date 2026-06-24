---
title: ".zshrcのチューニング: 203msから79msへ"
date: 2026-06-07T18:00:00+09:00
draft: false
ai_assisted: true
tags: ["zsh", "terminal", "dotfiles", "performance"]
categories: ["tech"]
description: "元記事に感化されてzshrcをいじった記録。フレームワークは捨てずに重複削除だけで2.5倍速になった話。"
---

## きっかけ

[Life is too short for a slow terminal](https://mijndertstuij.nl/posts/life-is-too-short-for-a-slow-terminal/) を読んだ。

とりあえず「自分のzshも計測してみるか」となった。

流石にTerminalとかGPUパワーでFPS改善するとかフレームワーク使うのやめるだとか、ガッツリオリジナルコード書きまくるほどではないにしても明確にコレは駄目だというものがあれば改善したい。

先程の記事の筆者はoh-my-zshもpreztoも使わない主義で30msを達成しているが、私はp10kのUIを捨てるコストは払いたくなかったので、フレームワーク（zinit + p10k）は維持したまま改善できる部分だけ潰す方針にした。

## まず計測

```zsh
time zsh -i -c exit
```

```
zsh -i -c exit  0.09s user 0.07s system 75% cpu 0.203 total
```

**203ms**。遅くはないが伸びしろがある気がする。

## zprof で犯人を特定する

`.zshrc` の先頭に：

```zsh
zmodload zsh/zprof
```

末尾に：

```zsh
zprof
```

を追加して新しいシェルを開くと、関数ごとの所要時間テーブルが出る。上位30件だけ見れば十分。

```zsh
zprof | head -n 30
```

```
num  calls                time                       self            name
-----------------------------------------------------------------------------------
 1) 1518         209.95     0.14   15.98%    153.70     0.10   11.70%  :zinit-tmp-subst-zle
 2)   60         107.81     1.80    8.20%     87.84     1.46    6.68%  _zsh_autosuggest_async_request
 3)    4         146.30    36.57   11.13%     67.91    16.98    5.17%  _zsh_autosuggest_bind_widgets
 4)  796          78.39     0.10    5.96%     67.84     0.09    5.16%  _zsh_autosuggest_bind_widget
 5)   34          98.12     2.89    7.47%     63.19     1.86    4.81%  -fast-highlight-process
 6) 2407          59.89     0.02    4.56%     59.89     0.02    4.56%  .zinit-add-report
...
```

`zinit-tmp-subst-zle` が1518回呼ばれていて1位。zinit がZLEウィジェットを差し替えるオーバーヘッドで、これはフレームワーク起因なので直接は触れない。

が、**calls数が異常に多い**のが気になった。

## 原因: zinit の2重ロード

`.zshrc` を見直すと、installer chunk の前後で `zinit.zsh` の source と `autoload` が重複していた。

```zsh
# installer chunk 内
source "$HOME/.zinit/bin/zinit.zsh"
autoload -Uz _zinit                    # ← 1回目
(( ${+_comps} )) && _comps[zinit]=_zinit

zinit light-mode for \
    zdharma-continuum/z-a-patch-dl \
    ...
### End of Zinit's installer chunk

# ↓ ここが余分
autoload -Uz _zinit                    # ← 2回目（重複）
(( ${+_comps} )) && _comps[zinit]=_zinit
zinit light zsh-users/zsh-autosuggestions
...
```

installer chunk をそのまま残しつつプラグインを追記したときに、`autoload` の行が二重になっていた。zinit が部分的に再初期化されてcalls数が倍増していた。

## 修正内容

重複していた2行を削除するだけ。

```zsh
# 削除した行
autoload -Uz _zinit
(( ${+_comps} )) && _comps[zinit]=_zinit
```

あわせて `history-search-multi-word` を eager load から turbo mode（wait/lucid）に変更した。

```zsh
# before
zinit load zdharma-continuum/history-search-multi-word

# after
zinit wait lucid for zdharma-continuum/history-search-multi-word
```

## 修正後の .zshrc（zinit 部分）

```zsh
source "$HOME/.zinit/bin/zinit.zsh"
typeset -g ZINIT[OPTIMIZE_OUT_DISK_ACCESSES]=1
autoload -Uz _zinit
(( ${+_comps} )) && _comps[zinit]=_zinit

zinit light-mode for \
    zdharma-continuum/z-a-patch-dl \
    zdharma-continuum/z-a-as-monitor \
    zdharma-continuum/z-a-bin-gem-node
### End of Zinit's installer chunk

zinit light zsh-users/zsh-autosuggestions
zinit wait lucid for zdharma-continuum/fast-syntax-highlighting
zinit wait lucid for zdharma-continuum/history-search-multi-word
zinit wait lucid atload"zicompinit; zicdreplay" blockf for zsh-users/zsh-completions
zplugin ice depth=1; zplugin light romkatv/powerlevel10k
```

## 結果

```zsh
time zsh -i -c exit
zsh -i -c exit  0.05s user 0.04s system 101% cpu 0.081 total
```

**203ms → 81ms**、約2.5倍速。

修正後の zprof 上位も calls 数が大幅に減少し、特定の支配的な犯人がいない状態になった。

## まとめ

- フレームワークを捨てなくても改善できる
- まず `time zsh -i -c exit` で計測、500ms超えてなければ優先度低め
- zprof で calls 数が異常に多い箇所を探す
- **重複ロードは見落としやすい**。installer chunk をコピペで使いまわすと起きがち
- `exec zsh` で設定を反映（`source ~/.zshrc` より確実）

地味な修正だったが、起動時間明確に早くなって嬉しい。体感あんまりわからんかも
