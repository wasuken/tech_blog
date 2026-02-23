---
title: "React NativeのTodoアプリで実装する相対時間ベースのプリセット機能"
date: 2026-02-08T10:50:00+09:00
draft: false
tags: ["React Native", "Expo", "TypeScript", "date-fns", "Todoアプリ", "設計パターン"]
categories: ["技術", "モバイル開発"]
description: "絶対時間ではなく相対時間（dueHoursOffset）で期限を管理することで、繰り返しタスクを効率化するプリセット機能の実装方法を解説"
---

## はじめに

Todoアプリを使っていると、毎日・毎週繰り返す定型タスクの登録が面倒に感じることはありませんか？

「毎朝のルーチン」「週次ミーティングの準備タスク」など、同じタスクセットを何度も手入力するのは非効率です。この記事では、**相対時間を使ったプリセット機能**の実装方法を紹介します。

実装したアプリのソースコード: https://github.com/your-repo (適宜修正してください)

## 問題：絶対時間で期限を保存すると使い回せない

一般的なTodoアプリでプリセット機能を実装する場合、以下のような設計になりがちです：

```typescript
// ❌ よくある実装（絶対時間）
interface PresetTask {
  text: string;
  dueDate: Date; // 2026-02-08 09:00:00
}
```

この設計の問題点：
- プリセット作成時の日時が保存される
- 翌日読み込むと「昨日の9時」が期限になってしまう
- 毎回手動で期限を修正する必要がある

## 解決策：相対時間（dueHoursOffset）で管理する

代わりに、**「今から何時間後」という相対的な時間**で期限を管理します：

```typescript
// ✅ 相対時間ベースの設計
export interface PresetTask {
  id: string;
  text: string;
  priority?: Priority;
  dueHoursOffset?: number; // 現在時刻からの相対時間（時間単位）
  checklist?: string[];
}

export interface Preset {
  id: string;
  name: string;
  tasks: PresetTask[];
  createdAt: Date;
}
```

**公式ドキュメント:**
- date-fns `addHours`: https://date-fns.org/v4.1.0/docs/addHours

## 実装の全体像

### 1. プリセット作成時の実装

プリセット編集画面では、期限を「現在時刻から何時間後」として入力します：

```typescript
// screens/PresetEditScreen.tsx
const TaskInputRow = ({
  item,
  index,
  onTaskTextChange,
  onDueHoursOffsetChange,
  // ...
}: {
  item: PresetTask;
  index: number;
  onTaskTextChange: (index: number, text: string) => void;
  onDueHoursOffsetChange: (index: number, value: string) => void;
  // ...
}) => {
  return (
    <Card style={styles.taskCard}>
      <View style={styles.taskInputRow}>
        <TextInput
          label={`タスク ${index + 1}`}
          value={item.text}
          onChangeText={text => onTaskTextChange(index, text)}
          mode="outlined"
          style={styles.taskTextInput}
          autoComplete="off"
          autoCorrect={false}
        />
        <TextInput
          label="期限(時間)"
          value={item.dueHoursOffset?.toString() || ''}
          onChangeText={value => {
            // 数字以外を除去
            const filteredValue = value.replace(/[^0-9]/g, '');
            onDueHoursOffsetChange(index, filteredValue);
          }}
          keyboardType="numeric"
          mode="outlined"
          style={styles.dueOffsetInput}
        />
      </View>
      {/* ... */}
    </Card>
  );
};
```

**ポイント:**
- `dueHoursOffset`に数値（時間）を入力
- 「24」と入力すれば「24時間後」という意味
- `keyboardType="numeric"`で数字キーボードを表示
- 数字以外の文字は`replace(/[^0-9]/g, '')`で除去

### 2. プリセット保存時の実装

入力されたデータをAsyncStorageに保存します：

```typescript
// screens/PresetEditScreen.tsx
const handleSave = async () => {
  if (!preset.name || preset.name.trim() === '') {
    setSnackbarVisible(true);
    return;
  }
  
  try {
    const storedPresets = await AsyncStorage.getItem(PRESETS_STORAGE_KEY);
    let presets: Preset[] = storedPresets ? JSON.parse(storedPresets) : [];

    // 空のタスクとチェックリスト項目を除去
    const tasks = (preset.tasks || [])
      .filter(t => t.text.trim() !== '')
      .map(t => ({
        ...t,
        checklist: (t.checklist || []).filter(c => c.trim() !== ''),
        priority: t.priority || Priority.Medium,
      }));

    if (isNew) {
      const newPreset: Preset = {
        id: Date.now().toString(),
        name: preset.name.trim(),
        tasks: tasks, // dueHoursOffsetがそのまま保存される
        createdAt: new Date(),
      };
      presets.push(newPreset);
    } else {
      presets = presets.map(p =>
        p.id === preset.id ? { ...p, name: preset.name!.trim(), tasks } : p
      );
    }

    await AsyncStorage.setItem(PRESETS_STORAGE_KEY, JSON.stringify(presets));
    navigation.goBack();
  } catch (error) {
    console.error('Failed to save preset.', error);
  }
};
```

**保存されるJSONの例:**
```json
{
  "id": "1738995600000",
  "name": "朝のルーチン",
  "tasks": [
    {
      "id": "task-1",
      "text": "メールチェック",
      "dueHoursOffset": 1,
      "priority": "高"
    },
    {
      "id": "task-2",
      "text": "日報作成",
      "dueHoursOffset": 8,
      "priority": "中"
    }
  ],
  "createdAt": "2026-02-08T00:00:00.000Z"
}
```

### 3. プリセット読み込み時の実装（核心部分）

保存されたプリセットを読み込んでタスクを生成する処理が、この機能の**最重要ポイント**です：

```typescript
// screens/PresetsScreen.tsx
import { addHours } from 'date-fns';

const handleLoadPreset = (tasks: PresetTask[]) => {
  const now = new Date(); // 現在時刻を取得
  
  tasks.forEach(presetTask => {
    let dueDate: Date | undefined = undefined;
    
    // dueHoursOffsetが設定されている場合のみ期限を計算
    if (presetTask.dueHoursOffset !== undefined) {
      dueDate = addHours(now, presetTask.dueHoursOffset);
    }
    
    // TodoContextのaddTodoを呼び出してタスクを追加
    addTodo(
      presetTask.text, 
      dueDate, 
      presetTask.checklist, 
      presetTask.priority
    );
  });
};
```

**動作の流れ:**

1. **プリセット読み込み時刻:** 2026-02-08 09:00
2. **タスク1（メールチェック）:** `dueHoursOffset: 1`
   - 期限 = `addHours(2026-02-08 09:00, 1)` = **2026-02-08 10:00**
3. **タスク2（日報作成）:** `dueHoursOffset: 8`
   - 期限 = `addHours(2026-02-08 09:00, 8)` = **2026-02-08 17:00**

**翌日に同じプリセットを読み込んだ場合:**

1. **プリセット読み込み時刻:** 2026-02-09 09:00
2. **タスク1（メールチェック）:** 期限 = **2026-02-09 10:00**
3. **タスク2（日報作成）:** 期限 = **2026-02-09 17:00**

**常に「今から」の相対時間で計算されるため、いつ読み込んでも適切な期限になります！**

### 4. date-fnsのaddHours関数について

```typescript
import { addHours } from 'date-fns';

const now = new Date('2026-02-08T09:00:00');
const future = addHours(now, 24);

console.log(future); // 2026-02-09T09:00:00
```

**date-fnsを選んだ理由:**
- サマータイム対応が正確
- タイムゾーン考慮が簡単
- TypeScript対応が優れている
- Tree-shakingでバンドルサイズ削減

**公式ドキュメント:**
- date-fns公式: https://date-fns.org/

## 実装の工夫ポイント

### 1. 期限なしタスクにも対応

```typescript
if (presetTask.dueHoursOffset !== undefined) {
  dueDate = addHours(now, presetTask.dueHoursOffset);
}
// dueHoursOffsetがundefinedなら、dueDateもundefinedのまま
```

期限を設定しないタスク（継続的なタスクなど）も扱えるようにしています。

### 2. 入力値のバリデーション

```typescript
onChangeText={value => {
  const filteredValue = value.replace(/[^0-9]/g, '');
  onDueHoursOffsetChange(index, filteredValue);
}}
```

数字以外の入力を除去することで、不正な値の保存を防いでいます。

### 3. チェックリストも一緒にプリセット化

```typescript
export interface PresetTask {
  id: string;
  text: string;
  priority?: Priority;
  dueHoursOffset?: number;
  checklist?: string[]; // ← チェックリストも含める
}
```

タスクに紐づくチェックリスト項目もプリセットに含めることで、より詳細なルーチンワークを定義できます。

## 応用例

### 1. 毎朝のルーチン

```typescript
{
  name: "朝のルーチン",
  tasks: [
    { text: "メールチェック", dueHoursOffset: 1 },      // 1時間後
    { text: "Slackの未読確認", dueHoursOffset: 1 },    // 1時間後
    { text: "午前のタスク整理", dueHoursOffset: 2 },   // 2時間後
    { text: "昼休憩", dueHoursOffset: 4 },              // 4時間後
  ]
}
```

### 2. 週次ミーティング準備

```typescript
{
  name: "週次ミーティング準備",
  tasks: [
    { text: "資料作成", dueHoursOffset: 48 },          // 2日後
    { text: "レビュー依頼", dueHoursOffset: 72 },      // 3日後
    { text: "最終確認", dueHoursOffset: 96 },          // 4日後
    { text: "ミーティング参加", dueHoursOffset: 120 }, // 5日後
  ]
}
```

### 3. プロジェクトキックオフ

```typescript
{
  name: "新規プロジェクト立ち上げ",
  tasks: [
    { text: "キックオフMTG", dueHoursOffset: 24 },
    { text: "要件定義", dueHoursOffset: 168 },         // 1週間後
    { text: "設計レビュー", dueHoursOffset: 336 },     // 2週間後
    { text: "実装開始", dueHoursOffset: 504 },         // 3週間後
  ]
}
```

## パフォーマンスの考慮

プリセット読み込み時に複数のタスクを一度に追加すると、再レンダリングが複数回発生する可能性があります。

現在の実装では、TodoContextの`addTodo`を複数回呼び出していますが、React 18の自動バッチングにより、実際の再レンダリングは1回にまとめられます：

```typescript
tasks.forEach(presetTask => {
  addTodo(...); // 複数回呼ばれても
});
// ↑ React 18が自動的に1回の再レンダリングにまとめる
```

**参考:**
- React 18 Automatic Batching: https://react.dev/blog/2022/03/29/react-v18#new-feature-automatic-batching

大量のタスク（100件以上）を一度に追加する場合は、バッチ追加用のAPIを実装することをおすすめします：

```typescript
// contexts/TodoContext.tsx に追加
const addTodoBatch = (tasks: Array<{
  text: string;
  dueDate?: Date;
  checklist?: string[];
  priority?: Priority;
}>) => {
  setTodos(prevTodos => [
    ...tasks.map(task => ({
      id: `${Date.now()}-${todoCounter++}-${Math.random()}`,
      text: task.text.trim(),
      status: 'todo' as TodoStatus,
      createdAt: new Date(),
      dueDate: task.dueDate,
      checklist: task.checklist?.map((item, index) => ({
        id: `${Date.now()}-cl-${index}-${Math.random()}`,
        text: item,
        completed: false,
      })) || [],
      priority: task.priority || Priority.Medium,
    })),
    ...prevTodos,
  ]);
};
```

## まとめ

相対時間ベースのプリセット機能を実装することで：

✅ **再利用性が高い**: いつ読み込んでも適切な期限が設定される  
✅ **直感的**: 「○時間後」という分かりやすい入力  
✅ **柔軟性**: 短期タスクから長期プロジェクトまで対応  
✅ **メンテナンス不要**: プリセット自体の更新が不要  

この設計パターンは、Todoアプリ以外にも応用可能です：
- リマインダーアプリ
- プロジェクト管理ツール
- 定期メンテナンススケジューラー

## 参考リンク

- **date-fns公式ドキュメント**: https://date-fns.org/
- **React Native Paper**: https://callstack.github.io/react-native-paper/
- **AsyncStorage**: https://react-native-async-storage.github.io/async-storage/

---

この実装について質問や改善案があれば、コメントでお知らせください！
