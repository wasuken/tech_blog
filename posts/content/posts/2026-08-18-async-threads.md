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
- `SharedPreferences`は外部ストレージへのI/Oが発生するため、頻繁な同期書き込みは避ける
- セルラー通信時にサイズの大きい処理を走らせる前は、ダイアログで一言断りを入れると親切。判定にはACCESS_NETWORK_STATE権限が必要になる
- エラーはその場で表示しつつ、後から確認できる形が理想。今ならCrashlytics/ACRAのようなクラッシュレポートツールで代替できる
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

今の主流はKotlin Coroutines(`viewModelScope`経由)やWorkManagerだ。Coroutinesはスレッドをブロックせず「中断するだけ」の軽量な並行処理で、排他制御が必要な場合は`Mutex`を使う。

### JVM側の大きな動き：Virtual Threads

Java側では、JDK 21でVirtual Threads(Project Loom)がJEP 444として正式機能になった。Java 19/20でプレビューされていたものが、Java 21で確定した形だ。従来のプラットフォームスレッドはOSスレッドと1:1で紐づいていたため生成コストが高かったが、Virtual Threadsは軽量なJVM管理スレッドとして大量生成が可能になる。I/Oバウンドな処理でリアクティブ系フレームワーク(WebFlux等)を置き換える流れが加速しているが、CPUバウンドな処理には向いておらず、そちらは引き続き`ExecutorService`の固定プールが有効という棲み分けになっている。

### Akka：分散システム向けに専門特化していった設計思想

Scala/JVM系のAkkaは、共有状態を持たせずアクター単位でメッセージパッシングする設計思想として知られている。ただし2022年9月、Akkaは全モジュールのライセンスをApache 2.0からBusiness Source License(BSL) v1.1へ変更した。年間売上が2,500万ドル未満の企業は無料の商用ライセンスで利用でき、それを超える企業は有償のサブスクリプションが必要になる。この変更以降、Akkaは汎用的な並行処理ライブラリというより、エンタープライズの分散システム向けに専門特化した領域として位置づけが変わったと見てよい。

### RN/Expo(JS)は前提が違う

React Native/Expoのようなシングルスレッド+イベントループの環境では、そもそも排他制御という概念自体が基本的には不要になる。Native Modules層はAndroid/iOS側のスレッドルールに従うため、JS側とネイティブ側で「排他制御が要るかどうか」の感覚が切り替わる点は意識しておきたい。

## 比較表：並行処理手段の選択肢

| 手段 | 排他制御 | 順序保証 | 管理単位 | 現在の位置づけ |
|---|---|---|---|---|
| `synchronized` | あり(モニターロック) | なし | ロック単位 | 低レベルの排他制御プリミティブとして現役 |
| Thread直接生成 | なし | なし(OS任せ) | スレッド単位 | 基本的には避け、上位の仕組みを使う |
| `ExecutorService` | 設計次第 | Single Thread Executorなら直列化で保証可 | プール+キュー | 現役、CPUバウンド処理で特に有効 |
| AsyncTask | なし | なし | Android固有の抽象 | API 30で非推奨、置き換え対象 |
| Kotlin Coroutines | `Mutex`で対応 | 設計次第 | 中断可能な軽量タスク | Android/Kotlinの主流 |
| Java Virtual Threads | 従来通り | 設計次第 | JVM管理の軽量スレッド | JDK 21以降、I/Oバウンド処理の主流候補 |
| Akka | アクターモデルで回避 | メッセージ順で保証可 | アクター単位 | BSLライセンス化以降、大規模分散システム向けに特化 |
| RN/Expo(JS) | 基本不要 | イベントループ順 | シングルスレッド | Native Modules層はネイティブのルールに従う |

## まとめ

- UXの土台は派手な最適化ではなく、`ProgressBar`表示やI/Oコストの管理といった地味な判断の積み重ねでできている
- `synchronized`は「同時に入らない」ことしか保証しない。順序を保証したいなら`Executors.newSingleThreadExecutor()`のような直列化の仕組みが要る
- `AsyncTask`はライフサイクル管理の弱さゆえにAPI 30で非推奨になり、今はCoroutinesやWorkManagerが主流
- JVM側はVirtual Threads(JDK 21)がI/Oバウンド処理の新しい選択肢として定着しつつある
- Akkaはライセンス変更以降、汎用ツールというより大規模分散システム向けの専門ツールという立ち位置に変わった

**「順序保証」と「排他制御」は似て非なるものであり、この区別を誤ると非同期処理の設計は必ずどこかで破綻する。**

## 参考

- [AsyncTask | Android Developers](https://developer.android.com/reference/android/os/AsyncTask)
- [Manage network usage | Connectivity | Android Developers](https://developer.android.com/develop/connectivity/network-ops/managing)
- [JEP 444: Virtual Threads](https://openjdk.org/jeps/444)
- [Why we are changing the license for Akka](https://akka.io/blog/why-we-are-changing-the-license-for-akka)
- [Akka BSL License FAQ](https://akka.io/bsl-license-faq)
