---
title: "react-native-pdf 6.7.7のiOS表示問題をpatch-packageで解決する"
date: 2026-02-17T14:30:00+09:00
tags: ["React Native", "iOS", "patch-package", "トラブルシューティング"]
---

## はじめに

業務で`react-native-pdf`を使用した際、AndroidではPDFが正常に表示されるのにiOSでは表示されないという問題に遭遇しました。

この記事では、GitHubのissueで共有された解決策である`patch-package`を使ったパッチ適用方法について解説します。

## 問題の概要

### 環境

```json
{
  "react-native-pdf": "^6.7.7",
  "react-native": "0.80.1",
  "react-native-blob-util": "^0.22.2"
}
```

### 症状

- Android: PDF表示が正常に動作
- iOS: PDFが表示されない

この問題は、React Native 0.80以降で`react-native-pdf`を使用した際に発生することが確認されています。

参考: [pdf is not displayed，Android is working fine, but there are problems with iOS #966](https://github.com/wonday/react-native-pdf/issues/966)

## 解決策: patch-packageを使う

GitHubのissueで[@anhnguyen123](https://github.com/wonday/react-native-pdf/issues/966)さんが共有してくれたパッチファイルを適用することで、この問題を解決できます。

### 1. patch-packageのインストール

まず、`patch-package`と`postinstall-postinstall`をdevDependenciesとしてインストールします。

```bash
# npmの場合
npm install --save-dev patch-package

# yarnの場合
yarn add --dev patch-package postinstall-postinstall
```

参考: [patch-package - npm](https://www.npmjs.com/package/patch-package)

### 2. package.jsonにpostinstallスクリプトを追加

`package.json`の`scripts`セクションに、`postinstall`スクリプトを追加します。

```json
{
  "scripts": {
    "postinstall": "patch-package"
  }
}
```

このスクリプトにより、`npm install`または`yarn install`を実行するたびに、自動的にパッチが適用されます。

### 3. パッチファイルの配置

GitHubのissueからパッチファイル`react-native-pdf+6.7.7.patch`をダウンロードし、プロジェクトルートに`patches`ディレクトリを作成してそこに配置します。

```
your-project/
├── patches/
│   └── react-native-pdf+6.7.7.patch
├── package.json
└── ...
```

### 4. パッチの適用確認

依存関係を再インストールして、パッチが正しく適用されることを確認します。

```bash
# node_modulesを削除して再インストール
rm -rf node_modules
npm install  # または yarn install
```

正常にパッチが適用されると、ターミナルに以下のようなメッセージが表示されます。

```
patch-package 8.0.0
Applying patches...
react-native-pdf@6.7.7 ✔
```

### 5. iOSのクリーンビルド

パッチ適用後は、iOSのビルドキャッシュをクリアしてから再ビルドします。

```bash
cd ios
rm -rf Pods Podfile.lock
pod install
cd ..

# キャッシュクリア
npx react-native start --reset-cache

# iOSビルド
npx react-native run-ios
# expo
npx expo run:ios
```

## postinstallとは何か

`postinstall`は、npmのライフサイクルスクリプトの一つで、`npm install`コマンドの実行後に自動的に実行されるスクリプトです。

### npmライフサイクルスクリプトの順序

```
preinstall → install → postinstall → prepublish → preprepare → prepare → postprepare
```

### postinstallの主な用途

1. **パッチの適用** (今回のケース)
   - `patch-package`を使った依存パッケージの修正

2. **ビルドステップの実行**
   - TypeScriptのコンパイル
   - ネイティブモジュールのビルド

3. **セットアップタスク**
   - 設定ファイルの生成
   - 環境の初期化

### postinstall-postinstallパッケージの役割

yarn v1では、`postinstall`スクリプトがサブディレクトリのパッケージに対して実行されないという制限があります。`postinstall-postinstall`パッケージは、この問題を回避するためのワークアラウンドです。

参考: [patch-package - Why use postinstall-postinstall](https://www.npmjs.com/package/patch-package#why-use-postinstall-postinstall)

## react-native-pdfはなぜパッチを当てる必要があるのか

### 主な理由

1. **React Nativeのバージョンアップへの追従遅れ**

React Nativeは頻繁にアップデートされますが、サードパーティライブラリの対応が追いつかないことがあります。`react-native-pdf`も例外ではありません。

2. **iOS/Androidのプラットフォーム固有の問題**

ネイティブコードを含むライブラリは、OS固有の問題に遭遇しやすく、特にiOSではビルドシステムやフレームワークの変更により互換性問題が発生します。

3. **メンテナンス状況**

GitHubのissueを見ると、375個のopenなissuesが存在している([issues](https://github.com/wonday/react-native-pdf/issues))

作者のGithubページを見る限り更新が完全に停止しており、更新頻度よりメンテナなどを立ていないようなので

新規プロジェクトなどはForkされたものなり代替パッケージを使ったほうが良さそう。

4. **具体的な技術的問題**

- React Native 0.78+での表示問題 ([#919](https://github.com/wonday/react-native-pdf/issues/919))
   - React Native 0.80 New Architectureとの互換性問題 ([#942](https://github.com/wonday/react-native-pdf/issues/942))
   - Expo SDK 54との互換性問題 ([#969](https://github.com/wonday/react-native-pdf/issues/969))

### パッチ適用のメリット・デメリット

#### メリット

- 即座に問題を解決できる
- フォークを作成する必要がない
- チーム全体で同じ修正を共有できる
- 公式の修正を待つ必要がない

#### デメリット

- ライブラリのバージョンアップ時に再度パッチが必要になる可能性
- 長期的なメンテナンスコスト
- 大規模な変更には不向き

参考: [patch-package - npm (When to use postinstall-postinstall)](https://www.npmjs.com/package/patch-package#when-to-use-postinstall-postinstall)

## 自分でパッチを作成する方法

GitHubで共有されているパッチが使えない場合や、独自の修正が必要な場合は、自分でパッチを作成できます。

### 手順

1. **`node_modules`内のファイルを直接編集**

```bash
# 例: iOS関連のファイルを修正
vim node_modules/react-native-pdf/ios/RCTPdf.m
```

2. **パッチファイルを生成**

```bash
npx patch-package react-native-pdf
```

これで`patches/react-native-pdf+6.7.7.patch`というファイルが自動生成されます。

3. **Gitにコミット**

```bash
git add patches/react-native-pdf+6.7.7.patch
git commit -m "fix: iOSでPDFが表示されない問題を修正"
```

4. **チームメンバーへの共有**

チームメンバーが`npm install`または`yarn install`を実行すると、自動的にパッチが適用されます。

参考: [Comprehensive Guide to Patching React Native Packages](https://medium.com/@yusufsancakk/comprehensive-guide-to-patching-react-native-packages-f7f73836ce6c)

## まとめ

- `react-native-pdf`のiOS表示問題は`patch-package`で解決できる
- `postinstall`スクリプトを使うことで、チーム全体で自動的にパッチを適用できる
- ライブラリのメンテナンス状況によっては、パッチ適用が現実的な解決策となる
- 長期的には公式の修正を待つか、代替ライブラリの検討も視野に入れる

## 参考リンク

- [react-native-pdf GitHub Issue #966](https://github.com/wonday/react-native-pdf/issues/966)
- [patch-package - npm](https://www.npmjs.com/package/patch-package)
- [patch-package - GitHub](https://github.com/ds300/patch-package)
- [react-native-pdf - npm](https://www.npmjs.com/package/react-native-pdf)
- [Comprehensive Guide to Patching React Native Packages - Medium](https://medium.com/@yusufsancakk/comprehensive-guide-to-patching-react-native-packages-f7f73836ce6c)
