---
title: "React NativeでTextInputの日本語入力が壊れる問題と解決方法"
date: 2026-01-31T18:30:00+09:00
draft: false
categories: ["React Native", "Expo"]
tags: ["React Native", "Expo", "TextInput", "IME", "バグ修正", "日本語入力"]
description: "React NativeのTextInputで日本語入力時に変換候補が消える問題の原因と、autoComplete/autoCorrectによる解決方法を解説します。"
---

## 問題：日本語入力で変換候補が消える

Expo/React Native で Todo アプリを作っていたとき、`TextInput` で日本語を入力すると変換候補が一瞬で消えてしまう問題に遭遇しました。
```tsx
// 問題のあるコード
const [text, setText] = useState('');

<TextInput
  value={text}
  onChangeText={setText}
/>
```

「あ」と入力しても変換候補が表示されず、即座に確定されてしまいます。ローマ字入力も正常に動作しません。

## 原因：State更新による再レンダリング

React Native の `TextInput` は制御コンポーネント（`value` + `onChangeText`）として使うと、以下の流れで問題が発生します：

1. 日本語入力で「あ」と入力
2. OS が変換候補を表示するために内部バッファを保持
3. `onChangeText` が発火して State 更新
4. 再レンダリングで `TextInput` が新しい value で再構築
5. **IME の内部バッファがリセットされる** ← ここが問題！
6. 変換候補が消える

`autoComplete` や `autoCorrect` が有効だと、OS の補完機能と React Native のブリッジ通信が複雑になり、IME との同期がズレやすくなります。

## 解決方法：autoComplete と autoCorrect を OFF
```tsx
// 修正後のコード
<TextInput
  value={text}
  onChangeText={setText}
  autoComplete="off"
  autoCorrect={false}
/>
```

この2つのプロパティを追加するだけで、IME が安定して動作するようになります。

### なぜこれで解決するのか？

- `autoComplete="off"`: OS の入力補完機能を無効化
- `autoCorrect={false}`: 自動修正機能を無効化

これらを OFF にすることで：
- ブリッジ通信（JavaScript ↔ Native）がシンプルになる
- IME の内部バッファ管理が安定する
- State 更新による再レンダリングの影響を受けにくくなる

## 代替案：defaultValue + onEndEditing

確実に動作させたい場合は、非制御コンポーネントにする方法もあります：
```tsx
<TextInput
  defaultValue={text}
  onEndEditing={(e) => setText(e.nativeEvent.text)}
/>
```

ただし、入力中の値がリアルタイムに取得できないため、UX が悪化する可能性があります。

## まとめ

React Native の `TextInput` で日本語入力が壊れる問題は、`autoComplete="off"` と `autoCorrect={false}` を追加するだけで解決できます。

これは React Native のブリッジ通信と IME の相性問題なので、将来のバージョンで改善される可能性もありますが、現時点ではこの対処法が最もシンプルで効果的です。

## 参考リンク

- [React Native TextInput 公式ドキュメント](https://reactnative.dev/docs/textinput)
- [Expo TextInput ガイド](https://docs.expo.dev/versions/latest/react-native/textinput/)
