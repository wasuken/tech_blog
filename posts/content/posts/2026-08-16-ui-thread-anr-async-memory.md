---
title: "Androidアプリ開発の極意 2,3章あたり ANR、UIスレッド"
date: 2026-08-16T18:00:00+09:00
draft: false
ai_assisted: true
tags: ["React Native", "Android", "iOS", "TypeScript", "パフォーマンス", "メモリ管理", "設計"]
categories: ["技術"]
description: "UIスレッド/ANRの基礎から、React NativeのJS-ネイティブスレッド分離、コルーチン/async-awaitの正体、Android/iOS/RNの非同期対応表、OOM/LMK/Android 17 Memory Limiter、Javaメモリリーク、useEffectクリーンアップまでを一気通貫で整理する（2026年8月時点の情報に更新）"
---

[Androidアプリ開発の極意 | 技術評論社](https://gihyo.jp/book/2017/978-4-7741-8817-1)

この本読んでる。2017年に出版してて、内容には古いものがいくつかある。

そこはAIの使いどころ。とりあえず学習したことを軽くメモして、中身について古い場合は指摘を入れたり、今は何が主流なのかといったことを補完するような指示を出した。

サポート付きの読書っぽいことをしながら読み進めて、文章をまとめた。

## 概要

React Native/Expoでアプリを書いていると、「なぜかカクつくが、クラッシュログには何も残っていない」という現象に何度もぶつかる。

原因を追っていくと、大体はUIスレッドを塞いでいるか、メモリを食い潰しているかのどちらかという同じ根っこに行き着くことが多い気がする。

この記事では、[Androidアプリ開発の極意 | 技術評論社](https://gihyo.jp/book/2017/978-4-7741-8817-1)の3章UIスレッド/ANRの基礎から、React NativeにおけるJS-ネイティブのスレッド分離、コルーチンとasync/awaitの正体、Android/iOS/RNの非同期処理対応、そして見落とされがちなメモリ管理（OOM・LMK・Javaのメモリリーク・useEffectのクリーンアップ）までを、実務で踏んだり踏みそうな地雷も交えて整理した。

なお、私は仕事でしばしReact Native Expoを利用しているため、なるべくそちらに沿った内容に寄せて本文を作成する。

それによって本の内容とは離れることがしばしあるが、そこは予め留意してほしい。

## 全体像

まずスレッドの分離構造をざっくり図にする。

```
[Android/ネイティブ]
  UIスレッド(メインスレッド) ── 描画・入力処理を担当
       │
       │ 5秒ブロック → ANRダイアログ
       ▼
  ワーカースレッド／コルーチン(Dispatchers.IO等) ── 重い処理はここへ逃がす

[React Native/Expo]
  JSスレッド ── ビジネスロジック・状態更新・レンダリング計算
       │
       │ (ブリッジ/JSI経由)
       ▼
  ネイティブUIスレッド ── 実際の描画
       │
  (任意) Worklet用の別JSコンテキスト ── react-native-worklets-core / Reanimated
```

ポイントは、**Androidのネイティブ層には「ANR」という可視化された強制終了の仕組みがあるが、React NativeのJSスレッドが詰まっても同じ仕組みは働かない**という非対称性にある。以下、順番に見ていく。

## UIスレッド/ANRの基礎

Androidでは、UIスレッド（メインスレッド）が入力イベントに応答できない状態が一定時間続くと、システムが強制的にANR（Application Not Responding）ダイアログを出す。Android公式ドキュメントによれば、入力イベント（キー押下や画面タップなど）に[5秒以内に応答しない場合にANRがトリガーされる。](https://developer.android.com/topic/performance/anrs/keep-your-app-responsive?hl=ja)これはフォアグラウンドの入力ディスパッチに関する条件で、他にもブロードキャストレシーバーやサービスの応答遅延など複数のANR条件が存在する。

一方、React Native/Expoでは事情が異なる。RNのアーキテクチャは**JSスレッドとネイティブUIスレッドが分離**されている。JSスレッドが重い処理で詰まっても、ネイティブ側のUIスレッド自体がブロックされているわけではないため、Android OSが検知する「入力ディスパッチのタイムアウト」という意味でのANRダイアログは出にくい。ただし、体感としては「タップしても反応がない」「アニメーションがカクつく」という、ユーザーから見れば**ANRと区別のつかない現象**が起きる。ANRダイアログが出ないぶん、むしろ発見が遅れて厄介とも言える。

> **補足**: 「RNならANRが原理的に起きない」という理解は不正確。JSスレッドが完全にブロックされている間、そのJSスレッドに紐づくタッチイベント処理やRedux的な状態更新も止まるため、ユーザー体験としては同種の「固まり」が発生する。OS側の検知メカニズムが働かないだけで、問題自体は消えていない。

## 重い処理の逃がし方：InteractionManager（旧API）とWorkletsは別物

RN側で「重い処理をUI/JSスレッドから逃がす」ための手段として、従来は `InteractionManager` がよく使われていたが、**2026年8月リリースのReact Native 0.87で `InteractionManager` は削除済み**であり、公式は代替として `requestIdleCallback` を案内している。0.82〜0.86の時点では非推奨警告が出るようになっていたため、既存プロジェクトでこの警告を見た人も多いはずだ。以降は「過去にどう動いていたか」の説明として読んでほしい。

### InteractionManager（旧API）：並列化ではなく「後回し」

`InteractionManager.runAfterInteractions()` は、旧公式ドキュメントの説明ではアニメーションや操作（インタラクション）が完了した後に長時間実行のタスクをスケジュールする仕組みだった。タッチ操作やアニメーションが「インタラクション」として扱われ、それが終わるまでコールバックの実行を遅延させる。既定では、キューに積まれたタスクは1回の setImmediate バッチでまとめて実行される仕組みで、実行自体はやはりJSスレッド上で行われていた。

つまり **InteractionManagerは「いつ実行するか」を制御するもので、「どこで実行するか」は変えていなかった**。アニメーション直後にCPUバウンドな処理を差し込むと、結局同じJSスレッド上でその処理がブロッキングとして効いてしまう。この性質は後継の `requestIdleCallback` でも変わらない。**「いつ実行するか」を制御する系統の手段はあくまでスケジューリングの調整であり、実行スレッド自体を変えるものではない**という理解が重要になる。

### Worklets：別ランタイム／別実行コンテキストへ処理を移せる

一方、Margeloが開発する `react-native-worklets-core` は、GitHubの説明にある通り、JS関数（Worklet）を別スレッドで実行するためのライブラリである。USAGE.mdでは、フィボナッチ計算のような重い処理をメインのJSスレッドをブロックせずにバックグラウンドスレッド上で実行できる例が示されている。

これとは別に、Software MansionのReanimated 4では、Worklet実装が `react-native-worklets` という独立ライブラリに分離されている。こちらは「複数のスレッド・ランタイム上でJavaScriptコードを並列実行できるライブラリ」と説明されており、`react-native-worklets-core` とは開発元・パッケージが異なる点に注意が必要だ。また、Worklet＝必ずバックグラウンドスレッドというわけでもなく、Reanimatedの世界ではUIスレッド側で動くWorkletも普通に存在する。正確には**「メインのJSランタイムとは異なる実行コンテキスト（別スレッド、あるいはUIスレッド上の別ランタイム）へ処理を移せる仕組み」**と捉えるのが実態に近い。

**「スケジューリングを後回しにするだけ」のInteractionManager／requestIdleCallbackと、「実行コンテキストそのものを変える」Worklets** ―― この違いを理解せずに「重いから後回しにしたのに変わらない」と嵌るケースは多い。

## コルーチン/async-awaitの正体

「コルーチン」という言葉が指すものは、サブプロセス（OSが新しいプロセスを起動する仕組み）とは全くの別レイヤーである。Kotlinの公式ドキュメントでもDispatchers.Mainはメインスレッドに紐づいたディスパッチャで、通常は単一スレッドで動作するとされており、コルーチンはスレッドそのものではなく、**スレッドの上で協調的にスケジューリングされる軽量なタスク単位**という位置づけになる。Android公式の解説でもDispatchers.Mainはコルーチンをメインのandroidスレッドで実行するためのディスパッチャで、UI操作や素早い処理にのみ使うべきとされている。

ここで正確に理解しておきたいのは、**async/await自体はI/O専用の仕組みではない**という点だ。async/awaitは「非同期処理の完了を待つための構文」であり、その処理が実際に別スレッド・別ワーカープールに投げられているなら、CPUバウンドな処理でも `await` できる（Kotlinの `withContext(Dispatchers.Default) { ... }` はまさにこの形）。

問題になるのは、次のようなコードだ。

```js
async function heavy() {
  // ここで巨大ループを回す
  for (let i = 0; i < 1e8; i++) { /* ... */ }
}
await heavy(); // asyncを付けても別スレッドには一切ならない
```

JavaScriptの `heavy()` を `async` 関数にしても、それだけでは処理が別スレッドに移るわけではなく、JSスレッド上で愚直にブロッキングする。**async/awaitは「待機中に実行スレッドを占有しない」設計と組み合わせやすいが、それ自体がCPUバウンドな処理を別スレッドへ移すわけではない**、という理解が正確になる。CPUバウンドな処理を本当に逃がすには、Kotlinなら `Dispatchers.Default` のような別スレッドプール、RNならWorkletsのような別実行コンテキストへ、処理そのものを移す必要がある。

## Android/iOS/RN 非同期処理 対応表

| 環境 | 主な機構 | UIスレッド保証の考え方 |
|---|---|---|
| Android (Kotlin) | `launch`/`async`, `suspend fun`, `Dispatchers.Main` | Main/Default/IOのディスパッチャを明示的に使い分ける |
| iOS (新, Swift Concurrency) | `Task`, `await`, `@MainActor` | `@MainActor`でメインスレッド（メインアクター）への隔離を宣言する |
| iOS (旧, GCD) | `DispatchQueue.main.async` | キュー単位でメイン/バックグラウンドを明示的に切り替える |
| JS (React Native) | `async function`, `await` | シングルスレッドが前提のため「UIスレッド保証」という概念自体が存在しない（JSスレッドは元々1つ） |

Swift Concurrencyにおける `@MainActor` は、メインスレッド上でタスクを実行するグローバルに一意なアクターであり、プロパティやメソッド、クロージャに付与することでメインスレッドでの実行を強制できる。GCD時代の `DispatchQueue.main.async` がキューベースだったのに対し、Swift Concurrencyはコンパイル時にスレッド安全性を検証できる点が大きな違いになる。

**JS行だけ「UIスレッド保証」の概念が存在しない、という非対称性がこの表の一番のポイントだ。** RNのJSスレッドはそもそも単一スレッドなので、「メインスレッドに切り替える」という操作自体が存在しない。その代わりに存在するのが、前述した「JSスレッド vs ネイティブUIスレッド」という別の軸の分離であり、ここを混同すると設計判断を誤る。

## 実務での遭遇パターン：SQLite後処理のCPUバウンド化

外部データ取得（API/DB呼び出し）自体は、async/awaitで素直に書けば大抵解決する。危険なのは**取得した後のデータ加工**だ。数万件規模のフィルタ・ソート・整形処理がJSスレッド上でCPUバウンドな処理として走ると、await で待っていたはずの非同期処理がその後段でみっちり固まる。

実体験としてANRっぽい落ち方に遭遇した原因も、まさにSQLiteから取得したデータをJS側で加工する処理がボトルネックだった。対策の優先度は次の順で考えるのが妥当だと思う。

1. **クエリ側で絞る**（`LIMIT`/`WHERE` でそもそも取得件数・加工対象を減らす）
2. **チャンク処理**（処理を小分けにして `setTimeout` や `requestIdleCallback` などでJSイベントループへ定期的に制御を返す。厳密に「1フレームあたりの処理量」を保証する仕組みではない点に注意）
3. **Worklets**（それでも足りなければ、別の実行コンテキストへ処理そのものを逃がす）

いきなりWorkletsに飛びつくより、まずクエリとチャンク分割で足りないかを確認する方がコストが低い。

なお、同種の重い処理を書いてもiOS側で同じ問題が発生しにくいと感じるのは、**端末スペックの裾野の広さの違い**（Androidは低スペック機の割合がiOSより高い）と、**ANRのような「固まり」を可視化する仕組みがOS側に存在しない**ことが理由として考えられる。これは公式ドキュメントで明言されている話ではなく、あくまで経験則としての整理である点は注記しておく。

## OOM と LMK：似て非なる2つの「落ち方」

メモリ関連のクラッシュには、大きく2種類ある。

- **OOM（OutOfMemoryError）**: アプリ自身のヒープが不足してクラッシュする。ログにスタックトレースが残る
- **LMK（Low Memory Killer）**: システム全体のメモリが逼迫したとき、OSが優先度の低いプロセスを黙って強制終了する

Android公式ドキュメントによれば、LMKデーモンは稼働中のAndroidシステムのメモリ状態を監視し、メモリ逼迫時に最も重要度の低いプロセスを終了させることでシステムのパフォーマンスを維持する仕組みで、oom_adj_score（カーネルレベルでは `oom_score_adj` とも呼ばれる）というスコアを使って優先度を判定し、スコアの高い（＝重要度の低い）プロセスから順に終了させる。この仕組みの厄介な点は、**Javaのヒープ不足によるOOMのような明確な例外・スタックトレースを残さず、他アプリの巻き添えでプロセスごと落とされる**ことにある。「ユーザーからは『アプリが落ちた』と報告が来るのに、クラッシュレポートにJavaの例外スタックトレースが残っていない」というケースでは、LMKが有力な候補の一つになる（ネイティブクラッシュやOSによる別種の強制終了など、他の原因もあり得るため「これしかない」と決め打ちはできない）。

> **補足**: Android 17（SDK 37）では、従来のLMKとは別に、デバイスの総RAM量に応じてプロセスごとのメモリ上限を設けるMemory Limiterという仕組みが公式に導入されている。cgroup v2ベースでプロセスのメモリ使用量を監視し、上限を超えたプロセスを終了させる。ここで注意したいのは、「ログが一切残らない」わけではないという点だ。Java側の`OutOfMemoryError`のようなスタックトレースは残らないものの、`ApplicationExitInfo.getDescription()` を確認すると `"MemoryLimiter"` という文字列が含まれる形で終了理由が記録される。「クラッシュログがないのにアプリが落ちる」系の原因調査では、LMKに加えてこのMemory Limiterも候補として押さえておきたい。

## Javaのメモリリークと対策

Javaのガベージコレクタは、「参照が生きている限り回収しない」という単純な原則で動く。この原則が牙を剥くのが、`static` フィールドやシングルトンがActivityへの参照を握り続けるパターンだ。Activityが破棄されても、どこかからの参照が生きている限りGCの対象にならず、メモリリークとして蓄積していく。

匿名クラスやリスナーが暗黙的に外側のActivityを参照してしまうパターンも同種の頻出事故で、リスナーの登録だけしてアンレジスタを書き忘れる、というのが典型例になる。対策としては次のようなものが定番になる。

- **WeakReference**: 強参照ではなく弱参照でActivityを保持し、GC対象から外れやすくする
- **リスナー解除の徹底**: 登録した箇所と対で、必ず解除処理を書く
- **ApplicationContext の使用**: Activity固有のContextではなく、アプリ全体で1つのContextを使うことでライフサイクルの罠を避ける
- **LeakCanaryでの検出**: 実際にリークが発生していないかを自動検出するツールを併用する

## RN/Expoのメモリ管理は別レイヤー

React Native/Expo環境では、JS層のオブジェクトはJSエンジン（Hermesなど）のGCが管理する。つまり、上述したような「Activity参照を握り続けて回収されない」というJava特有の問題は、JSレイヤーには**そのままの形では存在しない**。

画像コンポーネント（`expo-image` など）についても、キャッシュ管理はライブラリ側が担っており、基本的にアプリ側で意識する必要はない。ただし例外があり、**自作のネイティブモジュールを書く場合は、AndroidならJavaのメモリ管理ルール、iOSならARCのルールがそのまま適用される**。RN/Expoが管理してくれるのはあくまでJS層の話であり、ネイティブブリッジをまたいだ瞬間に前述のJavaのメモリリーク対策がそのまま必要になる、という点は見落とされやすい。

## useEffectのクリーンアップ = JS版のメモリリーク対策

`useEffect` の構造を整理すると次のようになる。

- **本体（①）**: 依存配列が変化したときに実行される処理（リスナー登録・購読開始など）
- **クリーンアップ（②、`return` で返す関数）**: 次回実行前、またはアンマウント時に呼ばれる処理（リスナー解除・購読停止など）
- **依存配列**: ①の再実行トリガー

```tsx
useEffect(() => {
  // ① 本体: リスナーを登録する
  const subscription = someEmitter.addListener('event', handler);

  // ② クリーンアップ: 次回実行前 or アンマウント時に呼ばれる
  return () => {
    subscription.remove();
  };
}, [handler]); // 依存配列: handlerが変わるたびに再実行
```

**リスナーを登録しておきながら `return` でのクリーンアップを書き忘れると、Java側のActivity参照リークと類似した構図がJS側でも発生する。** GCの管轄自体はHermesとJavaのGCで内部構造が異なるものの、「解放し忘れた参照がコンポーネントのライフサイクルを超えて生き続ける」という問題の形は共通している。RN/Expoのメモリ管理がJava的な罠から自由だからといって、useEffectのクリーンアップを疎かにしていい理由にはならない。

## まとめ

| テーマ | 要点 |
|---|---|
| UIスレッド/ANR | Androidは入力イベント5秒無応答でANR。RNのJSスレッドが詰まっても同じ検知機構は働かないが、体感は同じ |
| InteractionManager | RN 0.87で削除済み（旧API）。後継はrequestIdleCallbackだが、どちらも「後回し」であって実行スレッド自体は変えない |
| Worklets | 別の実行コンテキスト（別スレッド／別ランタイム）へ処理を移せる。CPUバウンドな重い処理はこちら |
| async/await | それ自体はI/O専用ではないが、CPUバウンドな処理を自動で別スレッドへ移すわけでもない |
| 実務の地雷 | 取得後のデータ加工がJSスレッド上でCPUバウンド化するケースが多い。クエリ絞り込み→チャンク処理→Workletsの順で検討 |
| OOM / LMK | 前者はヒープ不足でログが残る。後者はOS全体のメモリ逼迫で黙って落ちる |
| Javaメモリリーク | staticやリスナーがActivity参照を握り続ける構図。WeakReference・解除徹底・LeakCanaryで対処 |
| RN/Expo | JS層はHermesのGCが管理するが、自作ネイティブモジュールは各OSのルールがそのまま適用される |
| useEffect | クリーンアップの書き忘れは、Java版メモリリークのJS版と言える |

**「重い処理をどこで実行するか」と「参照をいつ手放すか」――この2つの設計判断を怠ると、プラットフォームが変わっても同じ形の障害が形を変えて再発する。**

## 参考

- [ANRs | App quality | Android Developers](https://developer.android.com/topic/performance/vitals/anr)
- [Keep your app responsive | App quality | Android Developers](https://developer.android.com/topic/performance/anrs/keep-your-app-responsive)
- [React Native 0.87 リリースブログ（InteractionManager削除の記載あり）](https://reactnative.dev/blog/2026/08/11/react-native-0.87)
- [InteractionManager · React Native（v0.81時点のドキュメント。旧API参照用）](https://reactnative.dev/docs/0.81/interactionmanager)
- [GitHub - margelo/react-native-worklets-core](https://github.com/margelo/react-native-worklets-core)
- [React Native Worklets: Multithreading engine for your apps and libraries](https://docs.swmansion.com/react-native-worklets/)
- [Migrating from Reanimated 3.x to 4.x（worklets分離の経緯）](https://docs.swmansion.com/react-native-reanimated/docs/guides/migration-from-3.x/)
- [Low memory killers | Android Developers](https://developer.android.com/games/optimize/vitals/lmk)
- [Low memory killer daemon | Android Open Source Project](https://source.android.com/docs/core/perf/lmkd)
- [Memory Limiter | Android Open Source Project（Android 17）](https://source.android.com/docs/core/perf/memory-limiter)
- [Prioritizing Memory Efficiency: Essential Steps for Android 17 | Android Developers Blog](https://android-developers.googleblog.com/2026/06/prioritizing-memory-efficiency-steps-for-android-17.html)
- [Improve app performance with Kotlin coroutines | Android Developers](https://developer.android.com/kotlin/coroutines/coroutines-adv)
- [Main | kotlinx.coroutines](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-main.html)
- [@MainActor in Swift explained with code examples](https://www.avanderlee.com/swift/mainactor-dispatch-main-thread/)
