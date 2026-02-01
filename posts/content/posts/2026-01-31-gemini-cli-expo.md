---
title: "Gemini CLIでExpo Todoアプリを爆速開発した話"
date: 2026-01-31T19:00:00+09:00
draft: false
categories: ["Expo", "AI開発"]
tags: ["Expo", "React Native", "Gemini", "AI", "Todo アプリ", "開発効率化"]
description: "Gemini CLI を使って Expo の Todo アプリを開発した体験記。GEMINI.md でルールを管理し、段階的に機能を追加していく方法を紹介します。"
---

## やりたかったこと

WSL2 環境で Expo を使った Todo アプリを作りたい。ただし、UI ライブラリの選定やナビゲーション設定など、細かい作業は Gemini に任せて効率化したい。

## GEMINI.md でルール管理

プロジェクトルートに `GEMINI.md` を作成し、Gemini に従ってほしいルールを記載しました。
```markdown
# Gemini AI Coding Rules

## Expo/React Native Specific
- Use Expo SDK compatible packages only
- Prefer `npx expo install` over `npm install`
- Use functional components with hooks

## Expo Specific Rules
- Use `@expo/vector-icons` instead of `react-native-vector-icons`
- Never use packages that require native linking

## Tech Stack (Fixed)
- Expo with TypeScript
- React Native Paper for UI
- AsyncStorage for persistence
```

このファイルを事前に作っておくことで、Gemini が一貫した品質のコードを生成してくれます。

## 段階的な開発

### Phase 1: 画面分割とナビゲーション
```bash
gemini "Update the todo app to add bottom tab navigation:

## Requirements
- Create 3 screens: Tasks, Presets, History
- Use React Navigation Bottom Tabs
- Make checkbox smaller and closer to text

Provide complete code for all files."
```

**ハマったポイント:**
- `react-native-vector-icons` ではなく `@expo/vector-icons` を使う必要があった
- タブアイコンの `route.name` が日本語タブ名と一致していなかった

解決策を GEMINI.md に追記：
```markdown
### Implemented Feature: Tab Bar Icons for Navigation

ハマったポイント:
- `route.name` の比較文字列を日本語タブ名に修正
- `@expo/vector-icons` の import 方法を明記
```

Gemini 自身に実装記録を追記させることで、次回以降同じミスを防げます。

### Phase 2: プリセット機能
```bash
gemini "Implement preset management feature in PresetsScreen:

## Requirements
- CRUD operations for presets
- Store in AsyncStorage key 'presets'
- Dialog for creating/editing preset
- Load button to add preset tasks to main list"
```

**ハマったポイント:**
- TextInput の日本語入力で変換候補が消える
- `autoComplete="off"` と `autoCorrect={false}` で解決

### Phase 3: カレンダーと履歴
```bash
gemini "Implement history and calendar view:

## UI Layout (Top-Bottom Split)
- Top 40%: Calendar with completion markers
- Bottom 60%: Completed task list for selected date

## Requirements
- Use react-native-calendars
- Save completedAt timestamp
- Mark dates with completed tasks"
```

**ハマったポイント:**
- MarkedDates の型定義が必要
- `List.Subheader` の配置位置（リストの最初に置く必要がある）

## GEMINI.md の自動更新

各フェーズ完了後、Gemini に実装記録を追記させます：
```bash
gemini "GEMINI.md の Phase 3 に実装記録を追記してください：

実装内容:
- react-native-calendars 使用
- completedAt フィールド追加

ハマったポイント:
- MarkedDates の型定義が必要
- List.Subheader の配置位置

Phase 1 と同じフォーマットで追記してください。"
```

これにより、GEMINI.md が開発ドキュメントとして自動的に成長していきます。

## トラブルシューティング：Jest 地獄

途中で Jest のテストを書こうとしましたが、`transformIgnorePatterns` の設定地獄にハマりました。
```
Cannot find module '@expo/vector-icons'
Cannot find module 'expo-asset'
Cannot find module 'expo-status-bar'
...
```

**結論:** 個人開発なら Jest は不要。E2E テスト（Maestro など）の方が実用的。

Jest を削除してスッキリしました。

## 成果物

約半日で以下の機能を実装：

- ✅ タスク追加・編集・削除・完了
- ✅ プリセット管理（定型タスクの一括登録）
- ✅ カレンダービューで完了履歴を確認
- ✅ React Native Paper による Material Design UI
- ✅ AsyncStorage によるデータ永続化

すべて Gemini CLI 経由で生成したコードです。

## Gemini CLI 開発のコツ

### 1. GEMINI.md でルールを事前定義

技術スタック、コーディング規約、出力フォーマットを明記しておく。

### 2. 段階的に機能追加

一度に全機能を依頼せず、Phase ごとに分割して実装。

### 3. ハマったポイントを記録させる

Gemini 自身に GEMINI.md へ追記させることで、同じミスを繰り返さない。

### 4. 具体的な指示を出す

曖昧な指示ではなく、データ構造、UI レイアウト、ファイル構成まで明示する。

## まとめ

Gemini CLI と GEMINI.md を使うことで、効率的に Expo アプリを開発できました。特に：

- UI ライブラリの選定や設定を任せられる
- ハマったポイントを記録して次に活かせる
- コード生成だけでなく、ドキュメント管理も自動化できる

次は Maestro で E2E テストを試してみる予定です。

## 参考リンク

- [Expo 公式ドキュメント](https://docs.expo.dev/)
- [React Native Paper](https://reactnativepaper.com/)
- [React Navigation](https://reactnavigation.org/)
- [react-native-calendars](https://github.com/wix/react-native-calendars)
