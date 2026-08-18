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

二部構成で一部は本を読んだ感想+α、二部がExpoだとどうなるかって話。

基本的にAIで出して、それを読んで調べたりしたもの。

# 第1部：Androidの話

Android(Java/Kotlin)側だけで完結する話。UXまわりの実装Tipsと、非同期処理の手段が`synchronized`からVirtual Threadsまでどう変遷してきたかを、Expoの話を挟まずに整理する。

## UXまわりの小さな判断

- 時間のかかる処理には`ProgressBar`で進捗を見せる
- `SharedPreferences`は永続化のためディスクI/Oが発生する
- セルラー通信時は重い処理の前に一言断りを入れると親切
- エラーはその場で表示しつつ、Crashlytics/ACRAのような開発者向けツールで後から追える形が理想
- NDKは高速化と引き換えにデバッグの複雑さが増す。基本的には奥の手

## 非同期処理はなぜ増え続けているのか

`synchronized`は「同時に入らない」ことしか保証しない。**順番までは保証しない。** Threadを直接生成した場合も同様で、生成順と実行順は一致しない。順番を保証したいなら`Executors.newSingleThreadExecutor()`のような直列化の仕組みを使う。

`ExecutorService`は、スレッドを直接管理せず、プールとキューで管理する仕組み。スレッド数の制御や再利用をアプリ側のロジックから切り離せる。

`AsyncTask`はAPI 30で非推奨になった。ライフサイクル管理が弱く、Activity/Fragment破棄後もタスクが動き続けてリークやクラッシュを招くのが理由。

今の主流はKotlin Coroutines。スレッドをブロックせず中断できる軽量な並行処理で、排他制御が必要なら`Mutex`を使う。UIに紐づく短い処理はCoroutines、画面を離れても続けるべき処理はWorkManagerと使い分ける。

JVM側ではJDK 21でVirtual Threads(JEP 444)が正式機能になった。I/Oバウンドな処理を大量にさばくのに向くが、CPUバウンドには向かない。ただしこれはJDK/Java SE側の機能で、Android Runtime(ART)は非対応。Androidアプリでそのまま使えるわけではない。

Akkaはアクター単位でメッセージパッシングする設計思想。2022年にライセンスがApache 2.0からBSL(Business Source License)に変わり、[2024年10月以降はproduction利用にライセンスキーが必須](https://akka.io/blog/akka-license-keys-and-no-spam-promise)になった。個人・OSS・startupは無料キー、商用は原則サブスクという体系。汎用ツールというより大規模分散システム向けの専門ツールという位置づけに変わっている。

## 比較表(Android)

| 手段 | 排他制御 | 順序保証 | 位置づけ |
|---|---|---|---|
| `synchronized` | あり | なし | 低レベルの排他制御プリミティブ |
| Thread直接生成 | なし | なし | 基本的には避ける |
| `ExecutorService` | 設計次第 | 直列化すれば可 | CPUバウンド処理で有効 |
| AsyncTask | なし | なし | API 30で非推奨 |
| Kotlin Coroutines | `Mutex`で対応 | 設計次第 | Android/Kotlinの主流 |
| Virtual Threads | 従来通り | 設計次第 | JVM側、I/Oバウンド向け(ART非対応) |
| Akka | アクターモデル | 条件付き | 大規模分散システム向け |

「排他制御」と「順序保証」は別物。ここを混同すると設計は破綻する。AsyncTaskはAPI 30で非推奨、今はCoroutines(短命処理)とWorkManager(永続処理)を使い分けるのが主流。Virtual Threads(JDK 21)はJVM側の話でARTは非対応。

# 第2部：Expoだとどうなるか

第1部で挙げたAndroidの各トピックが、Expo/React Nativeの世界では何に対応するか、あるいはそもそも関係なくなるかを、同じ順番でなぞっていく。

## UXまわりの小さな判断

- `ProgressBar`にあたるのは`ActivityIndicator`や`react-native-progress`のような表示ライブラリ。考え方自体は変わらない
- `SharedPreferences`に相当するのは、標準的なAsyncStorage(Expo Go対応・非同期API)。高頻度アクセスが絡む用途ではMMKV(同期API・高速だがネイティブビルド必須でExpo Go非対応)へ寄せる流れがある。認証トークンのような機密情報は`expo-secure-store`で分けるのが定石
- ネットワーク状態の確認は`expo-network`や`@react-native-community/netinfo`が担う
- エラー収集はSentryなどがCrashlytics/ACRA相当。ユーザー向け表示と開発者向け収集を分けて考えるのはAndroidと同じ
- NDKに近いのはNew Architecture(JSI/TurboModules)のC++コア。パフォーマンスとデバッグ容易性のトレードオフという構図は共通

## 非同期処理はなぜ増え続けているのか(Expo視点)

JS自体は単一スレッドで動くので、`synchronized`や`ExecutorService`のようなスレッドプール管理はJSコードの関心事にほぼならない。スレッドが登場するのはネイティブモジュール側の実装(Kotlin/Swift)で、そこでは第1部の話がそのまま使われる。

AsyncTask→Coroutinesという変遷も、JS側からは`await NativeModule.method()`と書くだけで意識せずに済む。裏側がCoroutinesかGCDかはモジュール実装次第。つまり非同期処理の実装品質はネイティブモジュール側に依存していて、自作モジュールを書くなら第1部の知見(AsyncTaskは避ける、Coroutinesを使う等)がそのまま活きる。

Virtual Threads(JDK 21)は、ARTが非対応な以上RNアプリの文脈でもほぼ関係ない。ネイティブ側の非同期処理は引き続きCoroutines/ExecutorServiceが選択肢になる。

RN/ExpoのJSはシングルスレッド+イベントループなので、mutexのような排他制御は基本不要。ただし非同期処理の完了順による論理的なrace conditionは別問題として存在する(`Promise.all`と逐次`await`では結果が変わる、など)。単一スレッド=常に安全、というわけではない。

## 比較表(Expo/RN)

| Androidの概念 | Expo/RNでの実質的な対応 |
|---|---|
| `SharedPreferences` | AsyncStorage / MMKV / expo-secure-store |
| Thread/`ExecutorService` | JS側は基本無関係。ネイティブモジュール実装側の話 |
| AsyncTask→Coroutines | JS側は`await NativeModule.method()`のみ |
| Virtual Threads | ARTが非対応のため基本関係ない |
| NDK | JSI/TurboModulesのC++コアが近い |

## まとめ

- Android側の非同期処理の変遷(synchronized→Thread→ExecutorService→AsyncTask→Coroutines→Virtual Threads)は「排他制御」と「順序保証」の違い、そして抽象化レベルの引き上げの歴史として整理できる
- Expo/RNではJSが単一スレッドなので、こうした話の多くがネイティブモジュールの裏側に隠れる。ただし非同期処理の完了順によるrace conditionは残るため、単一スレッド=無条件に安全ではない

**「排他制御」と「順序保証」は似て非なるもの。この区別を誤ると非同期処理の設計は必ずどこかで破綻する。**

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
