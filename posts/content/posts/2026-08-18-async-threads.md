---
title: "Androidアプリ開発の極意 4,5章ユーザー体験と非同期処理"
date: 2026-08-18T07:15:00+09:00
draft: false
ai_assisted: true
tags: ["Android", "並行処理", "Kotlin", "Java", "設計", "React Native"]
categories: ["技術"]
description: "synchronizedからVirtual Threadsまで、非同期処理の選択肢がどう変遷してきたかを整理する"
---

前の記事同様に

[Androidアプリ開発の極意 | 技術評論社](https://gihyo.jp/book/2017/978-4-7741-8817-1)の「ユーザーストレス軽減」まわりの実装Tipsと、並行処理の選択肢を整理してExpo周りと絡めて記事にしたもの。

## はじめに

「なぜアプリが固まるのか」を突き詰めると、大半はメインスレッド(UIスレッド)を長時間ブロックしていることに行き着く。逆に言えば、非同期処理まわりの設計は「ユーザーを待たせない」という一点のためにある。今回はUI/UX側の細かい実装Tipsと、非同期処理の手段がどう変遷してきたかを、まとめて整理する。

## 問題設定：待たせないための小さな判断の積み重ね

UXを損なわない実装は、派手な技術というより地味な判断の積み重ねでできている。

- 時間のかかる処理には`ProgressBar`で進捗を見せる(公式スタイルはあるが実装自体は各自で書く必要がある)
- 定数計算は`float`より`int`を優先し、頻出する計算は`static`ヘルパー関数に切り出す。ただし本来インスタンス前提で設計すべきものまで無闇に`static`化するのは悪手

  > **補足**: この数値型・static化の指針は、当時(2017年刊)の書籍の最適化Tipsとしては妥当だったが、現行のAndroid公式ドキュメントには「floatよりint」「static化を性能目的で推奨」という一般的な指針は見当たらない。現代ではまずプロファイリングして、実測に基づいて判断するのが基本になる。

- `SharedPreferences`は外部ストレージへのI/Oが発生するため、頻繁な同期書き込みは避ける

  > **認識ミス**: `SharedPreferences`が書き込むのは内部ストレージ上の永続ファイルで、「外部ストレージ」ではない。問題の本質はディスクI/O自体で、特に`commit()`による同期書き込みをUIスレッドで頻発させることが問題になる。`apply()`はメモリに即時反映してディスク書き込みを非同期化する。現在は用途によって[Preferences DataStore](https://developer.android.com/topic/libraries/architecture/datastore)も選択肢になる。

- セルラー通信時にサイズの大きい処理を走らせる前は、ダイアログで一言断りを入れると親切。判定にはACCESS_NETWORK_STATE権限が必要になる
- エラーはその場で表示しつつ、後から確認できる形が理想。今ならCrashlytics/ACRAのようなクラッシュレポートツールで代替できる

  > **補足**: Crashlytics/ACRAは基本的に開発者向けの障害検知手段であり、「ユーザーが後から確認できるエラー表示」そのものの代替にはならない。ユーザー向けにはその場の適切な表示、開発者向けにはCrashlytics/ACRAでの収集、と役割を分けて考えるのが正確。

- NDK(ネイティブコード)は高速化・自由度と引き換えにAPKサイズ増・デバッグ困難というトレードオフがあり、基本的には奥の手として扱う

これらはどれも単体では小さな話だが、積み重なるとアプリの「もっさり感」を決定づける。

## 本編：非同期処理の選択肢はなぜ増え続けているのか

### 排他制御と順序保証は別物

`synchronized`はモニターロックによる排他制御であり、「同時に入らない」ことは保証するが「実行される順番」までは保証しない。ここを混同すると「なぜかたまに結果が入れ替わる」という不具合の温床になる。

**Threadを直接生成した場合も同様で、生成順と実行順は一致しない。** スケジューリングはOS任せなので、`Print`程度の軽い処理ほど順序のブレが目立ちやすい。

順序を保証したい場合の選択肢は用途によって変わる。

- 並列化が不要なら、素直にメインループで書くのが一番安全
- Threadの形は保ちながら順序も保証したいなら、`Executors.newSingleThreadExecutor()`でプール1本+キューによる直列化を使う
- 並列に処理した後で結果だけ順番に並べ直す、という設計もあり得る

### ExecutorServiceが解決したこと

`ExecutorService`は、スレッドを直接管理するのではなく、あらかじめ確保したプール(作業員)とキュー(待ち行列)で管理する仕組みだ。内部的にはThreadを使っているが、利用者から見れば「Job処理としてのAPI」として扱える。この抽象化のおかげで、スレッド数の制御や再利用がアプリ側のロジックから切り離される。

### AsyncTaskはなぜ消えたのか

`AsyncTask`/`AsyncTaskLoader`はAndroid固有の仕組みだったが、API level 30で非推奨になった。理由はライフサイクル管理の弱さで、Activity/Fragmentが破棄された後もタスクが動き続けてリークやクラッシュを引き起こすことがあり、エラーハンドリングも弱かった。Android公式は代替として標準のjava.util.concurrentまたはKotlinの並行処理ユーティリティを使うよう案内している。

> **認識ミス**: `AsyncTask`本体の非推奨はAPI 30(Android 11)だが、framework版の[`AsyncTaskLoader`](https://developer.android.com/reference/android/content/AsyncTaskLoader)はAPI 28で非推奨になっており、時期が異なる。AndroidX版のLoaderは現在もAPI referenceが存在する。両者を同じタイミングとして扱うのは不正確。

今の主流はKotlin Coroutines(`viewModelScope`経由)やWorkManagerだ。Coroutinesはスレッドをブロックせず「中断するだけ」の軽量な並行処理で、排他制御が必要な場合は`Mutex`を使う。

> **補足**: CoroutinesとWorkManagerは用途が異なる。`viewModelScope`はUI/ViewModelのライフサイクルに紐づく比較的短い処理向けで、画面を離れても継続すべき永続的・遅延可能なバックグラウンド処理にはWorkManagerを使うのが公式の案内。また、Coroutineだから自動的にブロックしないわけではない。`suspend`可能な処理は待ち時間にスレッドを占有せず中断できるが、Coroutine内で通常のblocking I/Oを呼び出せば、そのスレッド自体は普通にブロックされる。

### JVM側の大きな動き：Virtual Threads

Java側では、JDK 21でVirtual Threads(Project Loom)がJEP 444として正式機能になった。Java 19/20でプレビューされていたものが、Java 21で確定した形だ。従来のプラットフォームスレッドはOSスレッドと1:1で紐づいていたため生成コストが高かったが、Virtual Threadsは軽量なJVM管理スレッドとして大量生成が可能になる。I/Oバウンドな処理でリアクティブ系フレームワーク(WebFlux等)を置き換える流れが加速しているが、CPUバウンドな処理には向いておらず、そちらは引き続きコア数を意識した`ExecutorService`/`ForkJoinPool`の固定プールが有効という棲み分けになっている。

> **補足**: ここで扱うVirtual ThreadsはAndroid Runtime(ART)の機能ではなく、JDK/Java SE側の話。Androidアプリの文脈でそのまま使えるわけではない点は注意したい。また、Virtual Threadsは並列に使えるCPUリソースそのものを増やす仕組みではなく、あくまで大量のblocking I/Oをthread-per-request型の素直なコードでスケールさせやすくする仕組みという理解が正確。

### Akka：分散システム向けに専門特化していった設計思想

Scala/JVM系のAkkaは、共有状態を持たせずアクター単位でメッセージパッシングする設計思想として知られている。ただし2022年9月、Akkaは全モジュールのライセンスをApache 2.0からBusiness Source License(BSL) v1.1へ変更した。年間売上が2,500万ドル未満の企業は無料の商用ライセンスで利用でき、それを超える企業は有償のサブスクリプションが必要になる。この変更以降、Akkaは汎用的な並行処理ライブラリというより、エンタープライズの分散システム向けに専門特化した領域として位置づけが変わったと見てよい。

> **補足**: 2022年のBSL移行はライセンス体系の出発点にすぎない。[2024年10月以降、新しいAkkaではproduction利用にライセンスキーが必要](https://akka.io/blog/akka-license-keys-and-no-spam-promise)になった。個人・OSS・startup等には無料キーが用意されている一方、商用プロジェクトでは原則としてサブスクリプションが必要になる。また、Akkaのメッセージ順序保証は特定のsender→receiver間などの条件付きであり、システム全体のglobal orderingを保証するものではない。

### RN/Expo(JS)は前提が違う

React Native/Expoのようなシングルスレッド+イベントループの環境では、そもそも排他制御という概念自体が基本的には不要になる。Native Modules層はAndroid/iOS側のスレッドルールに従うため、JS側とネイティブ側で「排他制御が要るかどうか」の感覚が切り替わる点は意識しておきたい。

> **補足**: JS実行自体は基本的に単一スレッドなので、CPUレベルの同時アクセスを防ぐmutexは通常不要というのは正しい。ただし非同期処理の完了順(`Promise.all`と逐次`await`の違いなど)による論理的なrace conditionは別問題として存在する。[React Native 0.76のNew Architecture](https://reactnative.dev/blog/2024/10/23/the-new-architecture-is-here)ではJSスレッド上のタスク処理順序を明確化する「well-defined event loop」が導入されており、単一スレッド=常に安全というわけではない。

## 比較表：並行処理手段の選択肢

| 手段 | 排他制御 | 順序保証 | 管理単位 | 現在の位置づけ |
|---|---|---|---|---|
| `synchronized` | あり(モニターロック) | なし | ロック単位 | 低レベルの排他制御プリミティブとして現役 |
| Thread直接生成 | なし | なし(OS任せ) | スレッド単位 | 基本的には避け、上位の仕組みを使う |
| `ExecutorService` | 設計次第 | Single Thread Executorなら直列化で保証可 | プール+キュー | 現役、CPUバウンド処理で特に有効 |
| AsyncTask | なし | なし | Android固有の抽象 | API 30で非推奨、置き換え対象 |
| Kotlin Coroutines | `Mutex`で対応 | 設計次第 | 中断可能な軽量タスク | Android/Kotlinの主流(用途によりWorkManagerと併用) |
| Java Virtual Threads | 従来通り | 設計次第 | JVM管理の軽量スレッド | JDK 21以降、I/Oバウンド処理の主流候補(JVM/Java SE側の話) |
| Akka | アクターモデルで回避 | sender→receiver等の条件付きで保証 | アクター単位 | BSL+ライセンスキー導入以降、大規模分散システム向けに特化 |
| RN/Expo(JS) | 基本不要(単一スレッド) | 非同期処理の完了順は別途設計が必要 | JSタスク/イベントループ | Native Modules層は別スレッドのルールに従う |

## まとめ

- UXの土台は派手な最適化ではなく、`ProgressBar`表示やI/Oコストの管理といった地味な判断の積み重ねでできている
- `synchronized`は「同時に入らない」ことしか保証しない。順序を保証したいなら`Executors.newSingleThreadExecutor()`のような直列化の仕組みが要る
- `AsyncTask`はライフサイクル管理の弱さゆえにAPI 30で非推奨になり、今はCoroutines(短命処理)とWorkManager(永続処理)を用途で使い分けるのが主流
- JVM側はVirtual Threads(JDK 21)がI/Oバウンド処理の新しい選択肢として定着しつつあるが、AndroidのARTの話ではない点に注意
- Akkaはライセンス変更以降、汎用ツールというより大規模分散システム向けの専門ツールという立ち位置に変わった

**「順序保証」と「排他制御」は似て非なるものであり、この区別を誤ると非同期処理の設計は必ずどこかで破綻する。**

## 参考

- [AsyncTask | Android Developers](https://developer.android.com/reference/android/os/AsyncTask)
- [AsyncTaskLoader | Android Developers](https://developer.android.com/reference/android/content/AsyncTaskLoader)
- [Preferences DataStore | Android Developers](https://developer.android.com/topic/libraries/architecture/datastore)
- [Manage network usage | Connectivity | Android Developers](https://developer.android.com/develop/connectivity/network-ops/managing)
- [JEP 444: Virtual Threads](https://openjdk.org/jeps/444)
- [Why we are changing the license for Akka](https://akka.io/blog/why-we-are-changing-the-license-for-akka)
- [Akka license keys and a no SPAM promise](https://akka.io/blog/akka-license-keys-and-no-spam-promise)
- [Akka BSL License FAQ](https://akka.io/bsl-license-faq)
- [New Architecture is here | React Native](https://reactnative.dev/blog/2024/10/23/the-new-architecture-is-here)
