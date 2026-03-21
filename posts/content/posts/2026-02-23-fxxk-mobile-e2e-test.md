---
title: "WSL2でExpo + E2Eテスト（MaestroとDetox）を試みて完全に詰んだ話"
date: 2026-02-23T21:30:00+09:00
tags: ["expo", "react-native", "detox", "maestro", "android", "wsl2", "e2e"]
---

## TL;DR

WSL2環境でExpo（React Native）のE2EテストをMaestroとDetoxで試みたが、どちらもWSL2とWindowsエミュレータの構造的な問題で動かなかった。

かなり過言ではあるが、あえて感情的になるならば、Mobile開発においてMac以外は人権がない。というかあまりにもMac環境以外がだるすぎる。

---

## 環境

- OS: Windows + WSL2（Ubuntu）
- Expo SDK 54 / React Native 0.81.5
- New Architecture有効
- Androidエミュレータ: Windows側で動作（Medium Phone API 36）
- ADB: Windows側のものをWSL2から参照

---

## Maestroを試みる

### インストール

```bash
curl -Ls "https://get.maestro.mobile.dev" | bash
export PATH="$HOME/.maestro/bin:$PATH"
```

ここで最初の罠。`maestro --help`を叩くとAI系の全く別のCLIツールが応答した。同名の別アプリが先にPATHに入っていたため。`$HOME/.maestro/bin`をPATHの**先頭**に置くことで解決。

### フローの準備

```yaml
# .maestro/add_and_complete_task.yml
appId: com.example.myapp
---
- launchApp
- tapOn:
    text: "追加"
- inputText: "テストタスク"
- tapOn:
    text: "追加する"
- assertVisible:
    text: "NOW"
```

### 実行して即死

```
You have 0 devices connected, which is not enough to run 1 shards.
```

エミュレータはWindows側で動いており、`adb devices`には`emulator-5554`が見えている。しかしMaestroはWSL2側でデバイスを探すため認識できない。

`--udid=emulator-5554`を指定しても：

```
Device emulator-5554 was requested, but it is not connected.
```

`maestro start-device --platform=android`を試みると：

```
This command is not supported in Windows WSL.
You can launch your emulator manually.
```

**公式が明言してWSL非対応。**

---

## Detoxを試みる

Maestroが詰んだのでDetoxに切り替え。

### セットアップ

```bash
npm i -D detox detox-cli jest @types/jest
npx detox init
```

### APKビルドでOutOfMemoryError

```
ERROR: D8: java.lang.OutOfMemoryError: Java heap space
```

`android/gradle.properties`に以下を追記して解決：

```properties
org.gradle.jvmargs=-Xmx4g -XX:MaxMetaspaceSize=512m
```

### `.detoxrc.js`の設定

AVD名を確認：

```bash
adb -s emulator-5554 emu avd name
# Medium_Phone_API_36.1
```

`android.emulator`タイプで設定後に実行すると：

```
There was no "emulator" executable file in directory: /home/user/Android/emulator
```

WSL2側にはAndroid SDKの`emulator`ディレクトリが存在しない。エミュレータはWindows側で動いているため当然。

### `android.attached`タイプに切り替え

エミュレータはすでに起動済みで`adb devices`に見えているので、`attached`タイプで接続を試みる：

```js
attached: {
  type: 'android.attached',
  device: {
    adbName: '.*'
  }
}
```

実行すると：

```
"adb" -s emulator-5554 shell "ps | grep \"com.example.myapp$\""
failed with error = Error: Command failed (code=1)
```

### 根本原因

DetoxはWSL2側のadbでエミュレータ内のプロセスを`ps | grep`で確認しようとする。しかし**エミュレータのプロセスはWindows側で動いている**ため、WSL2からは見えない。WebSocket接続も確立できずタイムアウト。

これはWSL2の構造上の問題であり、設定でどうにかなるものではなさそう。

---

## 結論：モバイル開発の人権マップ

| 環境 | Android E2E | iOS E2E |
|------|-------------|---------|
| **Mac** | ✅ 最強、何も制約なし | ✅ Xcodeも使える |
| **Linux (native)** | ✅ ギリいける | ❌ 物理的に不可 |
| **Windows (native)** | △ PowerShellで頑張ればいける | ❌ 物理的に不可 |
| **Windows + WSL2** | ❌ エミュレータ周りで詰む | ❌ 物理的に不可 |

WSL2は「Linuxっぽく使える」だけで「Linuxではない」。エミュレータのようにGPUやハードウェアアクセスが絡む処理は即死する。

---

## 現実的な代替案

### 1. Windows PowerShellでDetoxをネイティブ実行

WSLを捨てて、PowerShellにNode/npmを入れてそっちで実行する。エミュレータと同じWindows環境なので繋がる。

### 2. ユニットテストに留める

E2Eを諦めて、ロジック層（`hooks/`など）のみJestでユニットテストする。環境問題ゼロ。

### 3. CIでE2Eを動かす（EAS Workflows）

ローカルは諦めてCIでだけ動かす。EAS Workflowsには`type: maestro`のビルトインジョブがあり、設定が簡単。

しかしローカルでなくCIに任せるというのは・・・・。

```yaml
# .eas/workflows/e2e-android.yml
jobs:
  build:
    type: build
    params:
      platform: android
      profile: e2e
  maestro_test:
    needs: [build]
    type: maestro
    params:
      build_id: ${{ needs.build.outputs.build_id }}
      flow_path: ['.maestro/home.yml']
```

# 所感

Mobile開発をしたいならMac製品を買おう。
それ以外は買うなら苦労は覚悟しよう。

---

## 参考

- [Expo公式 EAS Workflows E2E](https://docs.expo.dev/eas/workflows/examples/e2e-tests/)
- [Maestro公式ドキュメント](https://docs.maestro.dev/)
- [Detox公式ドキュメント](https://wix.github.io/Detox/)
- [Maestro CLI インストール](https://docs.maestro.dev/maestro-cli/how-to-install-maestro-cli)
