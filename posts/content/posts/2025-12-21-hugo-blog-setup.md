---
title: "ローカル環境を汚さない静的サイト構築 - Hugo Docker Compose環境構築記録"
date: 2025-12-21T23:00:00+09:00
draft: false
tags: ["Hugo", "Docker", "静的サイトジェネレータ", "環境構築"]
---

## 背景

日課でできる範囲の活動として、軽い記事から、疑問を生成AIに出してもらって、それに答えてもらって、深堀や補足、添削をしてもらった内容までを記事にするという習慣を続けていたが、公開するのはどうなのかなと思った。

しかし、後ほど止めるのはもったいないということで妥協案として、ローカルで動くブログには投稿することにした。

なので、ローカルブログを立ち上げることにした。

最初はGitHub Pagesでよく使われているJekyllを試した。しかし、ローカル環境とDocker環境でRubyのバージョン不一致が発生し、プロジェクト初期化の段階で躓いた。

ローカルのRuby 3.4に対してDockerの最新イメージがRuby 3.1で、この差分が原因でSCSS変換周りでエラーが頻発。Jekyllはプロジェクト作成をローカルで行う必要があるため、「Docker使えば環境差を吸収できる」という謳い文句が実質的に機能しなかった。

もっとうまくやればよかっただろうが、そのときは血が登っていて、Hugoにしてしまった。

## 要件整理

改めて自分の要件を整理した：

- **Markdownファイルのマウントだけで完結**
- **ローカル環境に一切依存しない**
- **プロジェクト初期化もDocker内で実行可能**
- **検索機能とファイル一覧が欲しい**

これを満たすツールを探した結果、Hugoに行き着いた。

## なぜHugoなのか

Hugoを選んだ理由は明確：

### 1. バイナリ単体で動作

Go言語で書かれたHugoは単一バイナリで動作する。RubyやNode.js、Pythonのようなランタイム環境が不要。これにより依存関係地獄から解放される。

### 2. プロジェクト初期化もDocker内で完結

当初は生成AIの言うとおりに以下のコマンドでプロジェクトを作成した。

```bash
docker run --rm -v $(pwd)/posts:/src klakegg/hugo:alpine new site .
```

この1コマンドでプロジェクト作成が完了する。ローカルに何もインストールする必要がない。

のだが、後ほどこれがトラブルを産んだ。

### 3. 高速なビルド

Goの並列処理能力により、数千ページ規模のサイトでも秒単位でビルドが完了する。開発時のホットリロードも快適。

## 構築手順

### 1. docker-compose.yml作成

```yaml
services:
  hugo:
    image: hugomods/hugo:base
    container_name: hugo-blog
    ports:
      - "7000:7000"
    volumes:
      - ./posts:/src
    command: server --bind 0.0.0.0 --port 7000 --buildDrafts --buildFuture
    restart: unless-stopped
```

ポイント：

- `hugomods/hugo:base` を使用
- ポートは7000にマッピング（後述のブラウザ制限回避）
- `--buildDrafts --buildFuture` で下書きと未来日付の記事も表示

### 2. プロジェクト初期化

```bash
docker run --rm -v $(pwd)/posts:/src klakegg/hugo:alpine new site .
```

これで `posts/` ディレクトリに必要なファイル群が生成される。

のだが、ここは本来は

```bash
docker run --rm -v $(pwd)/posts:/src hugomods/hugo:base new site .
```

が正しいはず。私は一度間違えて、バージョン差異で一瞬止まったので注意。

### 3. テーマのインストール

検索機能と一覧表示が充実しているPaperModテーマを採用：

最初はanakeを試したが、シンプルすぎたのでPaperModへと変更。

```bash
cd posts
git clone https://github.com/adityatelange/hugo-PaperMod themes/PaperMod --depth=1
```

### 4. config.toml設定

```toml
baseURL = 'http://localhost:8080/'
languageCode = 'ja'
title = 'My Blog'
theme = 'PaperMod'

[params]
  ShowShareButtons = false
  ShowReadingTime = true
  ShowBreadCrumbs = true
  ShowPostNavLinks = true

[params.homeInfoParams]
  Title = "ブログ"
  Content = "技術メモ"

[[menu.main]]
  name = "アーカイブ"
  url = "/archives/"
  weight = 10
[[menu.main]]
  name = "検索"
  url = "/search/"
  weight = 20
[[menu.main]]
  name = "タグ"
  url = "/tags/"
  weight = 30

[outputs]
  home = ["HTML", "RSS", "JSON"]
```

### 5. 検索・アーカイブページ作成

```bash
mkdir -p posts/content

cat > posts/content/search.md << 'EOF'
---
title: "検索"
layout: "search"
---
EOF

cat > posts/content/archives.md << 'EOF'
---
title: "アーカイブ"
layout: "archives"
---
EOF
```

これ忘れてて404でて焦った。

### 6. 起動

```bash
docker compose up -d
```

http://localhost:7000 でアクセス可能。

## ハマったポイント

### ポート6000がブロックされる

最初ポート6000を指定したところ、Chrome/Edgeで `ERR_UNSAFE_PORT` エラーが発生。

**原因**: ポート6000はX11関連で予約されており、Chromiumベースのブラウザがセキュリティ上ブロックするみたいだ。

**解決**: ポートを6000以外(ここでは7000)に変更して解決。

参考: [Chromium Blocked Ports](https://chromium.googlesource.com/chromium/src.git/+/refs/heads/main/net/base/port_util.cc)

### テーマなしでは何も表示されない

Hugoはテーマが必須。テーマを入れないと `page not found` になる。

最初 `klakegg/hugo:alpine` イメージを使用したが、バージョンが古く（v0.111.3）、最新のテーマと互換性がなかった。`hugomods/hugo:base` に変更することで解決。

## 記事の配置

Markdownファイルは `posts/content/posts/` に配置：

```
posts/
├── content/
│   ├── posts/
│   │   ├── 2025-12-21-first-post.md
│   │   └── 2025-12-22-second-post.md
│   ├── search.md
│   └── archives.md
├── themes/
│   └── PaperMod/
└── config.toml
```

記事のフォーマット例：

```markdown
---
title: "記事タイトル"
date: 2025-12-21T10:00:00+09:00
draft: false
tags: ["タグ1", "タグ2"]
---

本文をここに書く
```

## PaperModの検索機能

PaperModテーマはFuse.jsを使った全文検索を内蔵している。`config.toml` で `[outputs]` に `JSON` を追加することで、検索用のインデックスが自動生成される。

検索ページ（`/search/`）にアクセスすると、リアルタイムで記事をフィルタリングできる。完全にクライアントサイドで動作するため、サーバーサイドの実装は不要。

## まとめ

完全にDocker内で完結する静的サイト構築環境をHugoで実現できた。

**利点**:

- ローカル環境を一切汚さない
- プロジェクト作成から起動まで全てDocker内で完結
- 高速なビルドと快適な開発体験
- 検索・一覧機能も標準的なテーマで実現可能

**注意点**:

- テーマは必須（完全ゼロからの構築は手間）
- Dockerイメージのバージョン選定が重要
- ブラウザの安全でないポート制限に注意

静的サイトジェネレータは他にもZola（Rust製）やAstro（Node.js）など選択肢があるが、バイナリ単体で動作し、Dockerとの親和性が高いHugoは「環境を汚したくない」要件に最適だった。

## 参考

- [Hugo公式ドキュメント](https://gohugo.io/documentation/)
- [HugoMods Docker Image](https://docker.hugomods.com/docs/)
- [PaperMod テーマ](https://github.com/adityatelange/hugo-PaperMod)
