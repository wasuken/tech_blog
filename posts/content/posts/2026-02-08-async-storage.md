---
title: "AsyncStorageって裏側何やってんの？ - 2.0と3.0の実装の違いを調べてみた"
date: 2026-02-08T17:50:00+09:00
draft: false
tags: ["React Native", "Expo", "AsyncStorage", "永続化"]
---

私は普段React NativeでExpo触ってるので、AsyncStorageはよく使うんだけど、「そういえばAsyncStorageって裏側何やってんだろう？」って疑問が湧いてきたので調べてみることにした。

## AsyncStorageの裏側

AsyncStorageのバージョンによって実装が少し違う。

### AsyncStorage 2.0の実装

iOS/Androidのみ調査。

公式ドキュメント: [Where your data is stored - Async Storage](https://react-native-async-storage.github.io/2.0/advanced/Where-data-stored/)

#### iOS (2.0)

- `manifest.json`ファイルに保存される
- JSONファイル形式
- パス: `Documents/RCTAsyncLocalStorage_V1/manifest.json`
- 詳細: 1024文字以下のデータは`manifest.json`に、それより大きいデータは個別ファイル(MD5ハッシュ名)に保存される

#### Android (2.0)
- SQLiteデータベースに保存される
- データベース名: `RKStorage`
- パス: `/data/data/{package_name}/databases/RKStorage`

### AsyncStorage 3.0 (next)の実装

**公式ドキュメント**: https://react-native-async-storage.github.io/3.0-next/

#### 対応プラットフォーム

- Android (SQLite)
- iOS (SQLite) ✨
- macOS (SQLite)
- visionOS (legacy fallback, single database only)
- Web (IndexedDB backend)
- Windows (legacy fallback, single database only)

##### iOS (3.0)

- SQLiteデータベースに変更された
- Androidと同じ実装に統一
- パフォーマンスと安定性が向上

##### Android (3.0)

- 引き続きSQLite
- より洗練された実装

3.0からはiOSもAndroidも両方SQLiteになって、実装が統一されるそうだ。

##### 互換性

- React Native 0.76以降が必要(iOS/Android)
- Kotlin 2.1.0
- iOS minimum target: 13
- Android minimum SDK: 24

## なぜiOSでmanifest.jsonからSQLiteに変更したのか

あくまでも推測ではあるがやってみた。

### manifest.jsonの問題点

#### パフォーマンス

- JSON全体を読み込む必要がある
- データが大きくなると起動時の読み込みが遅い
- 部分的な読み書きができない

#### 並行処理

- ファイルロックの問題
- 複数の書き込みが競合しやすい

#### データ整合性

- JSON書き込み中にクラッシュするとデータが壊れる可能性

### manifest.jsonで実際に起きた問題

#### Issue #88

週に約1,500件の頻度で「The folder "manifest.json" doesn't exist」というクラッシュが報告されていました。

参考URL: [The folder “manifest.json” doesn’t exist · Issue #88 · react-native-async-storage/async-storage · GitHub](https://github.com/react-native-async-storage/async-storage/issues/88)

#### Issue #897

Documentsフォルダに配置されるため、ユーザーがファイルアプリから機密情報を含むmanifest.jsonにアクセス可能でした。

参考URL: [iOS RCTAsyncLocalStorage_V1 folder still showing under Documents · Issue #897 · react-native-async-storage/async-storage · GitHub](https://github.com/react-native-async-storage/async-storage/issues/897)

### SQLiteのメリット

**パフォーマンス**:
- 必要なキーだけ読み込める
- インデックスが効く
- 大量データでも高速

**トランザクション**:
- ACID特性が保証される
- データ破損のリスクが低い

**並行処理**:
- SQLiteはロック機構が優秀
- 複数スレッドからのアクセスに強い

だから3.0で統一したんだろう。合理的な判断だと思う。

## Expo SDKとAsyncStorageのバージョン

Expoを使ってる場合、AsyncStorageのバージョンはExpo SDKに依存する。

私が今使ってるExpo SDK（54）だと、AsyncStorage 2.0系が入ってるはず。

つまり：

- iOSは`manifest.json`
- Androidは`SQLite`

という非対称な状態。

3.0はまだ`next`だから、正式リリースされたら両方SQLiteになる。

## AsyncStorage 3.0のインストール

### npmの場合
```bash
npm install @react-native-async-storage/async-storage@next
```

### yarnの場合
```bash
yarn add @react-native-async-storage/async-storage@next
```

### Androidの追加設定

`android/build.gradle`に以下を追加:
```gradle
allprojects {
    repositories {
        // ... others like google(), mavenCentral()

        maven {
            url = uri(project(":react-native-async-storage_async-storage").file("local_repo"))
        }
    }
}
```

### iOSの追加設定
```bash
cd ios
pod install
```

### 使い方
```typescript
import { createAsyncStorage } from "@react-native-async-storage/async-storage";

// create a storage instance
const storage = createAsyncStorage("appDB");

async function demo() {
  await storage.setItem("userToken", "abc123");

  const token = await storage.getItem("userToken");
  console.log("Stored token:", token); // abc123

  await storage.removeItem("userToken");
}
```

## AsyncStorage 2.0と3.0の移行

3.0に移行する時、データは自動で移行されるのか？

公式ドキュメント見た感じ、マイグレーション処理は入ってそう。ただ、大量データがある場合は移行に時間かかるかもしれない。

まあ、Expoが3.0対応した時に確認する感じか。

## AsyncStorageのデータ破損対策

AsyncStorage 2.0のiOS実装（manifest.json）だと、データ破損のリスクがある。

実際、Issueとかで「AsyncStorageのデータが壊れた」って報告をたまに見る。

対策

1. 重要なデータは複数のキーに分散して保存
2. バックアップ用のキーを別途用意
3. 読み込み時にJSON.parseのエラーハンドリングをちゃんとする
4. できれば3.0に移行する
5. そもそもSqliteとかRealmを独自で使う

```typescript
try {
  const jsonString = await AsyncStorage.getItem('important_data');
  const data = jsonString ? JSON.parse(jsonString) : null;
  return data;
} catch (error) {
  console.error('AsyncStorage parse error:', error);
  // バックアップから復元を試みる
  const backupString = await AsyncStorage.getItem('important_data_backup');
  if (backupString) {
    try {
      return JSON.parse(backupString);
    } catch (backupError) {
      // 諦める
      return null;
    }
  }
  return null;
}
```

## ExpoでAsyncStorageを使う時の注意点

### 1. @react-native-async-storage/async-storageを使う

公式のAsyncStorageはなくなったのでコミュニティ版を使う。
```bash
npx expo install @react-native-async-storage/async-storage
```

### 2. バージョンを確認する

2.0系なのか3.0系なのかで実装が違う。
```bash
npx expo install @react-native-async-storage/async-storage --check
```

### 3. JSONのシリアライズ/デシリアライズは自分でやる

AsyncStorageは文字列しか保存できない。
```typescript
// 保存
const data = { foo: 'bar', count: 42 };
await AsyncStorage.setItem('key', JSON.stringify(data));

// 読み込み
const jsonString = await AsyncStorage.getItem('key');
const data = jsonString ? JSON.parse(jsonString) : null;
```

### 4. multiGet/multiSetを使う

複数のキーを一度に読み書きする時は、multiGet/multiSetを使うと効率的。
```typescript
// 一つずつだと遅い
const value1 = await AsyncStorage.getItem('key1');
const value2 = await AsyncStorage.getItem('key2');

// まとめて読む
const values = await AsyncStorage.multiGet(['key1', 'key2']);
// [['key1', 'value1'], ['key2', 'value2']]
```

### 5. 大量データは避ける

AsyncStorageは大量データには向いてない。

目安：
- 数百件のキー・バリューペアまで
- 1キーあたり数KB~数十KB程度

それ以上なら、WatermelonDBとかRealmとか、ちゃんとしたデータベース使った方が良い。

## 結論

AsyncStorageの裏側、調べてみたら思ってたより複雑だった。

### AsyncStorage 2.0

- iOS: manifest.json（JSON）
- Android: SQLite
- 非対称な実装

### AsyncStorage 3.0

- iOS/Android: 両方SQLite
- 実装が統一される
- パフォーマンスと安定性が向上


Expoで3.0が使えるようになったら、積極的に移行したい。SQLiteに統一されるのは良いことだと思う。

## 参考文献

- [AsyncStorage 2.0 - Where data is stored](https://react-native-async-storage.github.io/2.0/advanced/Where-data-stored/)
- [AsyncStorage 3.0-next Documentation](https://react-native-async-storage.github.io/3.0-next/)
- [AsyncStorage Issue #88 - manifest.json folder error](https://github.com/react-native-async-storage/async-storage/issues/88)
- [AsyncStorage Issue #897 - iOS RCTAsyncLocalStorage_V1 folder](https://github.com/react-native-async-storage/async-storage/issues/897)
