---
title: "React NativeでTextInputの日本語入力が壊れる問題と解決方法"
date: 2026-01-31T18:30:00+09:00
draft: false
categories: ["React Native", "Expo"]
tags: ["React Native", "Expo", "TextInput", "IME", "バグ修正", "日本語入力"]
description: "React NativeのTextInputで日本語入力時に変換候補が消える問題の原因と、autoComplete/autoCorrectによる解決方法を解説します。"
---

最近趣味の方でモバイル開発を始めた。
Android端末を普段遣いしている点、仕事上iOSのアプリ周りのリリースがクソだるいことを知っているため
Expoで開発しつつも、Androidのみを想定した開発を行っている。
その延長線で引っかかった部分とかをメモに残そうと思ったので記事にした。

## 問題：日本語入力で変換候補が消える

Expo/React Native で Todo アプリを作っていたとき`TextInput` で日本語を入力すると変換候補が一瞬で消えてしまう問題に遭遇。

```tsx
// 問題のあるコード
const [text, setText] = useState('');
<TextInput
  value={text}
  onChangeText={setText}
/>
```

「あ」と入力しても変換候補が表示されず、即座に確定されてしまい、ローマ字入力も正常に動作しない。

## 原因：State更新による再レンダリング

React Native の `TextInput` は制御コンポーネント（`value` + `onChangeText`）として使うと、以下の流れで問題が発生する。

1. 日本語入力で「あ」と入力
2. OS が変換候補を表示するために内部バッファを保持
3. `onChangeText` が発火して State 更新
4. 再レンダリングで `TextInput` が新しい value で再構築
5. **controlled component としての `value` の強制が、IME の内部バッファと衝突する** ← ここが問題！
6. 変換候補が消える

`autoComplete` や `autoCorrect` が有効だと、OS の補完機能が `value` の強制にさらに抵抗するため、IME との同期がズレやすくなる。

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

この2つのプロパティを追加するだけで、IME が安定して動作した。

### なぜこれで解決するのか？

- `autoComplete="off"`: OS の入力補完機能を無効化し、`value` の強制に対する抵抗を減らす
- `autoCorrect={false}`: 自動修正機能を無効化する。**ただし、Androidでは実質無効であることが報告されている**（[GitHub issue #18457](https://github.com/facebook/react-native/issues/18457)）

したがって、Android上で問題が解決したとすれば、実際には **`autoComplete="off"` だけが有効だった可能性が高い**。`autoCorrect={false}` を倣って追加しておくのは、将来的にiOS対応を行った時のためのものとして捉えてよい。

## 代替案：defaultValue + onEndEditing

リアルタイム同期を放棄してよい場合は、非制御コンポーネントにする方法もある。

```tsx
<TextInput
  defaultValue={text}
  onEndEditing={(e) => setText(e.nativeEvent.text)}
/>
```

この場合、`value` による強制が発生しないため、IME のバッファリセットは起こらない。
ただし、入力中の値がリアルタイムに取得できないため、UX が悪化する可能性があったり、管理しているデータの状態によっては解決しないこともあるので注意。

## まとめ

私が遭遇した問題に関してはReact Native の `TextInput` で日本語入力が壊れる問題は`autoComplete="off"` と `autoCorrect={false}` を追加するだけで解決できた。
ただし、Androidでは `autoCorrect={false}` が実質無効であるため、実際には `autoComplete="off"` が主たる解決策であった可能性が高い。
これは React Native の controlled component としての `value` 強制と IME の相性問題かもしれない。
またいつか別の記事にするかもしれないが、前述した通りデータの構成によってはこれらを追加しても意味がないこともある。
あくまでもアプローチの一つとして捉えてほしい。

## 参考リンク

- [React Native TextInput 公式ドキュメント](https://reactnative.dev/docs/textinput)
- [autoCorrect Android で無効になっていない報告 #18457](https://github.com/facebook/react-native/issues/18457)
- [autoCorrect のデフォルト値がAndroidで異なる報告 #20063](https://github.com/facebook/react-native/issues/20063)
