---
title: "GEMINI.mdでAIに開発履歴を管理させる方法"
date: 2026-01-31T19:30:00+09:00
draft: false
categories: ["AI開発", "開発効率化"]
tags: ["Gemini", "AI", "ドキュメント管理", "開発手法", "プロジェクト管理"]
description: "GEMINI.mdファイルを使ってAI（Gemini）にコーディングルールと開発履歴を管理させる手法を紹介します。AIが自分で学習・改善していくドキュメント駆動開発の実践例。"
---

## 問題：AIは過去の失敗を忘れる

Gemini や ChatGPT などの AI にコード生成を依頼するとき、こんな問題がありませんか？

- 同じミスを何度も繰り返す
- 前回指摘したルールを忘れる
- プロジェクト固有の制約を無視する

毎回「Expo では react-native-vector-icons じゃなくて @expo/vector-icons を使って」と指示するのは面倒です。

## 解決策：GEMINI.md でルールを管理

プロジェクトルートに `GEMINI.md` ファイルを作成し、AI に従ってほしいルールをすべて記載します。
```markdown
# Gemini AI Coding Rules

## General Principles
- Always provide complete, working code
- Include all necessary imports
- Add TypeScript types for all functions

## Expo/React Native Specific
- Use Expo SDK compatible packages only
- Prefer `npx expo install` over `npm install`

## Expo Specific Rules
- Use `@expo/vector-icons` instead of `react-native-vector-icons`
- Import example: `import { MaterialCommunityIcons } from '@expo/vector-icons';`
```

AI に指示を出すときは、必ず「GEMINI.md を読んでから実装して」と伝えます。
```bash
gemini "GEMINI.md を読んでから、タブナビゲーションを実装してください"
```

## GEMINI.md の構成

### 1. 基本ルール

すべてのプロジェクトで共通のルールを記載。
```markdown
## General Principles
- Always provide complete, working code
- No placeholder code like "// Add your logic here"
- Include error handling

## Output Format
When generating code, always include:
1. Complete file content (not snippets)
2. Installation commands for dependencies
3. File path/name where code should be placed
```

### 2. プロジェクト固有のルール

技術スタックやデータ構造を明記。
```markdown
## Tech Stack (Fixed)
- Expo with TypeScript
- React Native Paper for UI
- AsyncStorage for persistence

## Data Model
\`\`\`typescript
interface Todo {
  id: string;
  text: string;
  completed: boolean;
  createdAt: Date;
}
\`\`\`
```

### 3. フェーズ管理

段階的な開発計画を記載。
```markdown
# Phase 1: Navigation & UI Improvements

## Requirements
- Use React Navigation Bottom Tabs
- 3 tabs: Tasks, Presets, History
- Make checkboxes smaller
```

## AI 自身に履歴を追記させる

ここが重要なポイントです。**実装完了後、AI 自身に GEMINI.md へ記録を追記させます。**
```bash
gemini "GEMINI.md の Phase 1 セクションに、今回実装した内容を追記してください：

実装内容:
- タブナビゲーション実装
- @expo/vector-icons 使用

ハマったポイント:
- route.name が日本語タブ名と一致していなかった
- iconName の型を厳密に指定する必要があった

Phase 1 の実装記録として追記してください。"
```

すると、AI が以下のような記録を追記してくれます：
```markdown
### Implemented Feature: Tab Bar Icons for Navigation

- **概要**: タブナビゲーションを実装。日本語タブ名に対応したアイコン表示。

- **使用したライブラリとバージョン**:
    - `@react-navigation/bottom-tabs`: `^7.10.1`
    - `@expo/vector-icons`: (Expo SDK に含まれる)

- **ハマったポイントと解決策**:
    - **ハマったポイント**: `route.name` が日本語タブ名と一致していなかった
    - **解決策**: 比較文字列を日本語に修正し、型を厳密に指定

- **次回への引き継ぎ事項**:
    - デフォルトアイコンの UI/UX 検討が必要
```

## GEMINI.md のメリット

### 1. 同じミスを繰り返さない
```markdown
## Expo Specific Rules
- Use `@expo/vector-icons` instead of `react-native-vector-icons`
```

一度ルールに追加すれば、AI は二度と間違えません。

### 2. プロジェクトの歴史が残る
```markdown
# Phase 1: Completed (2026-01-31)
# Phase 2: Completed (2026-01-31)
# Phase 3: Completed (2026-01-31)
```

いつ何を実装したか、どんな問題があったかが一目瞭然。

### 3. 新しい開発者へのオンボーディング

GEMINI.md を読めば：
- プロジェクトの技術スタック
- 過去にハマったポイント
- 開発の進行状況

がすべてわかります。人間の開発者にとっても有用なドキュメントになります。

### 4. AI が自己学習する

実装記録を AI 自身に書かせることで、次回の実装時に「前回はこうハマったから気をつけよう」という判断ができるようになります。

## 実践例：3 つのフェーズで開発

### Phase 1: ナビゲーション
```bash
gemini "GEMINI.md を読んで、Phase 1 を実装してください"
```

実装後：
```bash
gemini "Phase 1 の実装記録を GEMINI.md に追記してください"
```

### Phase 2: プリセット機能

Phase 1 の記録があるので、AI は：
- `@expo/vector-icons` を使う
- TypeScript の型を厳密に定義する

などを自動的に考慮してくれます。

### Phase 3: カレンダー

Phase 1, 2 の記録から：
- `react-native-calendars` のインストール方法
- 型定義の必要性
- UI コンポーネントの配置

などを学習済み。

## GEMINI.md のテンプレート
```markdown
# Gemini AI Coding Rules

## General Principles
- (共通ルール)

## Tech Stack
- (使用技術)

## Data Model
- (データ構造)

---

# Phase 1: (機能名)

## Requirements
- (要件)

## Implementation
- (実装内容)

### Implemented Feature: (実装した機能)

- **概要**: 
- **使用したライブラリとバージョン**:
- **ハマったポイントと解決策**:
- **次回への引き継ぎ事項**:

---

# Phase 2: (次の機能)
...
```

## 注意点

### 1. AI はファイル編集できないことがある
```bash
gemini "GEMINI.md に追記してください"
```

と指示しても、**出力だけして実際にファイルを編集しない**場合があります。

その場合は：
- AI の出力をコピペして手動で追記
- または「完全なファイル内容を出力してください」と指示して置き換え

### 2. ファイルパスを明示する
```bash
gemini "./GEMINI.md の Phase 3 に追記してください"
```

相対パスを明示すると、AI がファイルを特定しやすくなります。

### 3. フォーマットを統一する

Phase 1 で使ったフォーマットを Phase 2, 3 でも使うよう指示：
```bash
gemini "Phase 1 と同じフォーマットで Phase 3 の記録を追記してください"
```

## まとめ

GEMINI.md を使うことで：

- ✅ AI が同じミスを繰り返さない
- ✅ プロジェクトの歴史が自動的に記録される
- ✅ 開発効率が大幅に向上する
- ✅ ドキュメント管理が自動化される

AI を単なるコード生成ツールではなく、**学習・改善していくチームメンバー**として扱えるようになります。

次回のプロジェクトでは、ぜひ GEMINI.md を試してみてください。

## 参考

今回の Todo アプリ開発の GEMINI.md は以下のような構成になりました：

- General Principles
- Expo Specific Rules
- Phase 1: Navigation (実装記録付き)
- Phase 2: Preset Management
- Phase 3: History & Calendar

約 200 行のドキュメントが、AI と協力しながら自動的に育っていきました。
