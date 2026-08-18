---
title: "Androidアプリ開発の極意 4,5章ユーザー体験と非同期処理"
date: 2026-08-18T07:15:00+09:00
draft: false
ai_assisted: true
tags: ["Android", "並行処理", "Kotlin", "Java", "設計", "React Native"]
categories: ["技術"]
description: "synchronizedからCoroutinesまで、非同期処理の選択肢がどう変遷してきたかを軽く整理する"
---

前の記事同様に

[Androidアプリ開発の極意 | 技術評論社](https://gihyo.jp/book/2017/978-4-7741-8817-1)の「ユーザーストレス軽減」まわりの実装Tipsと、並行処理の選択肢をざっくり整理してExpo周りと絡めたもの。細かい実装には踏み込まず、要点だけ。

## はじめに

アプリが固まる原因の大半は、メインスレッド(UIスレッド)を長時間ブロックしていること。非同期処理の設計は結局「ユーザーを待たせない」という一点のためにある。

## UXまわりの小さな判断

- 時間のかかる処理には`ProgressBar`で進捗を見せる
- `SharedPreferences`は永続化のためディスクI/Oが発生する

  > **認識ミス**: 「外部ストレージへのI/O」という表現は誤り。実際は内部ストレージ上のファイルへの書き込みで、問題の本質はディスクI/O自体。`commit()`は同期、`apply()`は非同期。現在は用途によって[DataStore](https://developer.android.com/topic/libraries/architecture/datastore)も選択肢。

  Expoでは、AsyncStorage(標準的・Expo Go対応)やMMKV(高速だがネイティブビルド必須)がこれにあたる。

- セルラー通信時は重い処理の前に一言断りを入れると親切
- エラーはその場で表示しつつ、Crashlytics/ACRAのような開発者向けツールで後から追える形が理想
- NDKは高速化と引き換えにデバッグの複雑さが増す。Expoで言えばNew Architecture(JSI/TurboModules)のC++コアが近い

## 非同期処理はなぜ増え続けているのか

`synchronized`は「同時に入らない」ことしか保証しない。**順番までは保証しない。** Threadを直接生成した場合も同様で、生成順と実行順は一致しない。順番を保証したいなら`Executors.newSingleThreadExecutor()`のような直列化の仕組みを使う。

`ExecutorService`は、スレッドを直接管理せず、プールとキューで管理する仕組み。スレッド数の制御や再利用をアプリ側のロジックから切り離せる。

Expoでは、JS自体は単一スレッドなのでこの手の話はほぼ登場しない。スレッドプールが出てくるのはネイティブモジュール側の実装(Kotlin/Swift)だけ。

`AsyncTask`はAPI 30で非推奨になった。ライフサイクル管理が弱く、Activity/Fragment破棄後もタスクが動き続けてリークやクラッシュを招くのが理由。

> **認識ミス**: framework版`AsyncTaskLoader`はAPI 28で非推奨と、AsyncTask本体(API 30)とは時期が異なる。同一視するのは不正確。

今の主流はKotlin Coroutines。スレッドをブロックせず中断できる軽量な並行処理で、排他制御が必要なら`Mutex`を使う。UIに紐づく短い処理はCoroutines、画面を離れても続けるべき処理はWorkManagerと使い分ける。

Expoでは、JS側は`await NativeModule.method()`と書くだけで、裏側がCoroutinesかGCDかはモジュール次第で意識しなくてよい。

JVM側ではJDK 21でVirtual Threads(JEP 444)が正式機能になった。I/Oバウンドな処理を大量にさばくのに向くが、CPUバウンドには向かない。

> **補足**: これはJDK/Java SE側の話で、Android Runtime(ART)は非対応。Androidアプリでそのまま使えるわけではない。Expoでも同様に基本関係ない。

Akkaはアクター単位でメッセージパッシングする設計思想。2022年にライセンスがApache 2.0からBSL(Business Source License)に変わり、[2024年10月以降はproduction利用にライセンスキーが必須](https://akka.io/blog/akka-license-keys-and-no-spam-promise)になった。個人・OSS・startupは無料キー、商用は原則サブスクという体系。汎用ツールというより大規模分散システム向けの専門ツールという位置づけに変わっている。

RN/ExpoのJSはシングルスレッド+イベントループなので、mutexのような排他制御は基本不要。ただし非同期処理の完了順による論理的なrace conditionは別問題として存在する(`Promise.all`と逐次`await`では結果が変わる、など)。

## 比較表

| 手段 | 排他制御 | 順序保証 | 位置づけ |
|---|---|---|---|
| `synchronized` | あり | なし | 低レベルの排他制御プリミティブ |
| Thread直接生成 | なし | なし | 基本的には避ける |
| `ExecutorService` | 設計次第 | 直列化すれば可 | CPUバウンド処理で有効 |
| AsyncTask | なし | なし | API 30で非推奨 |
| Kotlin Coroutines | `Mutex`で対応 | 設計次第 | Android/Kotlinの主流 |
| Virtual Threads | 従来通り | 設計次第 | JVM側、I/Oバウンド向け(ART非対応) |
| Akka | アクターモデル | 条件付き | 大規模分散システム向け |
| RN/Expo(JS) | 基本不要 | 完了順は別途設計 | Native Modules層は別ルール |

## まとめ

- UXは派手な最適化より、I/Oコストのような地味な判断の積み重ねでできている
- 「排他制御」と「順序保証」は別物。ここを混同すると設計は破綻する
- AsyncTaskはAPI 30で非推奨、今はCoroutines(短命処理)とWorkManager(永続処理)を使い分けるのが主流
- Virtual Threads(JDK 21)はJVM側の話でARTは非対応
- Expoでは非同期処理の多くがネイティブモジュールの裏側に隠れるが、実装の質はAndroid側の知見に依存する

**「順序保証」と「排他制御」は似て非なるもの。この区別を誤ると非同期処理の設計は必ずどこかで破綻する。**

## 参考

- [AsyncTask | Android Developers](https://developer.android.com/reference/android/os/AsyncTask)
- [AsyncTaskLoader | Android Developers](https://developer.android.com/reference/android/content/AsyncTaskLoader)
- [Preferences DataStore | Android Developers](https://developer.android.com/topic/libraries/architecture/datastore)
- [JEP 444: Virtual Threads](https://openjdk.org/jeps/444)
- [Why we are changing the license for Akka](https://akka.io/blog/why-we-are-changing-the-license-for-akka)
- [Akka license keys and a no SPAM promise](https://akka.io/blog/akka-license-keys-and-no-spam-promise)
- [New Architecture is here | React Native](https://reactnative.dev/blog/2024/10/23/the-new-architecture-is-here)
- [react-native-async-storage/async-storage | GitHub](https://github.com/react-native-async-storage/async-storage)
- [mrousavy/react-native-mmkv | GitHub](https://github.com/mrousavy/react-native-mmkv)
