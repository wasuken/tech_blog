---
title: "歴史地図アプリに日本語検索を実装: GeoJSONデータの効率的な翻訳手法"
date: 2026-02-15T19:00:00+09:00
tags: ["GeoJSON", "Gemini API", "翻訳", "地図アプリ", "データ拡張"]
---

## はじめに

歴史的国境を可視化する地図アプリを作っていたら、「日本語で国名検索ができない」という問題に直面した。外部のGeoJSONデータは英語のみで、日本語プロパティがない。

そこで、**Gemini APIを使って効率的にデータを翻訳し、日本語検索を実装した**手法を紹介する。

## 問題: 外部GeoJSONデータには日本語がない

使用したデータソース:
- **現代国境**: [Natural Earth](https://www.naturalearthdata.com/) (約200カ国)
- **歴史的国境**: [aourednik/historical-basemaps](https://github.com/aourednik/historical-basemaps) (18ファイル、紀元前2000年〜1920年)

```json
{
  "type": "Feature",
  "properties": {
    "NAME": "France",
    "NAME_JA": null  // ← 日本語プロパティがない!
  },
  "geometry": { ... }
}
```

このままでは「フランス」で検索できない。

## 解決策: 翻訳キャッシュを使った効率的なデータ拡張

### アプローチ1: 愚直な方法 (非効率)

各ファイルごとに全データをLLMに投げる:

```javascript
// ❌ 非効率: 同じ国名を何度も翻訳
for (const file of geoJsonFiles) {
  const data = await fetch(file);
  const translated = await translateAll(data); // Franceを18回翻訳...
  await save(translated);
}
```

**問題点:**
- 同じ国名が複数ファイルに登場 → 重複翻訳
- トークン消費が膨大
- 処理時間が長い

### アプローチ2: 翻訳キャッシュ方式 (効率的) ✅

**全ファイル共通の翻訳キャッシュを使い回す:**

```javascript
// ✅ 効率的: 一度翻訳した国名は二度と翻訳しない
const translationCache = {}; // { "France": "フランス", ... }

for (const file of geoJsonFiles) {
  const data = await fetch(file);
  
  // 未翻訳の国名のみ抽出
  const newNames = extractUntranslatedNames(data, translationCache);
  
  // 新規の国名だけ翻訳
  if (newNames.length > 0) {
    const translations = await translate(newNames);
    Object.assign(translationCache, translations);
  }
  
  // キャッシュを使って適用
  applyTranslations(data, translationCache);
  await save(data);
}
```

## 実装: Node.jsスクリプト

### 完全なコード

この手法をNode.jsスクリプトとして実装した。Gemini 2.5 Flash Liteを使用している。この程度の翻訳ならこれで十分。

```javascript
#!/usr/bin/env node

import fs from 'fs/promises';
import path from 'path';
import { fileURLToPath } from 'url';

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

const HISTORICAL_BASE_URL = 'https://raw.githubusercontent.com/aourednik/historical-basemaps/master/geojson';

/**
 * Gemini Flash APIで翻訳
 */
async function translateToJapanese(countryNames) {
  const apiKey = process.env.GEMINI_API_KEY;
  if (!apiKey) {
    throw new Error('GEMINI_API_KEY環境変数が設定されていません');
  }

  const prompt = `以下の国名・地域名を日本語に翻訳してください。
歴史的な国家名も含まれているため、適切な日本語表記を選んでください。

出力形式: JSONオブジェクト { "英語名": "日本語名", ... }
注意: 
- 翻訳できない場合は空文字列""を返す
- 歴史的な国名も考慮する(例: "Roman Empire" -> "ローマ帝国")
- JSONのみを出力し、説明文は不要

国名リスト:
${JSON.stringify(countryNames, null, 2)}`;

  const url = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-lite:generateContent?key=${apiKey}`;
  
  const response = await fetch(url, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      contents: [{ parts: [{ text: prompt }] }],
      generationConfig: {
        temperature: 0.2,
        topK: 40,
        topP: 0.95,
        maxOutputTokens: 8192,
      }
    })
  });

  if (!response.ok) {
    const errorText = await response.text();
    throw new Error(`Gemini API error: ${response.status}\n${errorText}`);
  }

  const data = await response.json();
  const content = data.candidates[0].content.parts[0].text;
  
  // JSONを抽出
  let jsonText = content.trim();
  if (jsonText.startsWith('```')) {
    jsonText = jsonText.replace(/^```(?:json)?\n?/, '').replace(/\n?```$/, '');
  }
  
  const jsonMatch = jsonText.match(/\{[\s\S]*\}/);
  if (!jsonMatch) {
    throw new Error('JSON形式ではありません');
  }

  return JSON.parse(jsonMatch[0]);
}

/**
 * GeoJSONに日本語名を追加
 */
async function addJapaneseNames(geojson, translationCache = {}) {
  const features = geojson.features;
  
  // 未翻訳の国名を収集
  const untranslatedNames = [];
  for (const feature of features) {
    const name = feature.properties.NAME;
    if (name && !translationCache[name]) {
      untranslatedNames.push(name);
    }
  }

  // 新規の翻訳のみAPI呼び出し
  if (untranslatedNames.length > 0) {
    const uniqueNames = [...new Set(untranslatedNames)];
    
    // バッチサイズ100で処理
    const batchSize = 100;
    for (let i = 0; i < uniqueNames.length; i += batchSize) {
      const batch = uniqueNames.slice(i, i + batchSize);
      console.log(`  📦 バッチ ${Math.floor(i / batchSize) + 1}/${Math.ceil(uniqueNames.length / batchSize)} (${batch.length}件)`);
      
      try {
        const translations = await translateToJapanese(batch);
        Object.assign(translationCache, translations);
        
        // レート制限対策
        if (i + batchSize < uniqueNames.length) {
          await new Promise(resolve => setTimeout(resolve, 500));
        }
      } catch (error) {
        console.error(`  ❌ エラー:`, error.message);
      }
    }
  }

  // 翻訳を適用
  let appliedCount = 0;
  for (const feature of features) {
    const name = feature.properties.NAME;
    if (name && translationCache[name]) {
      feature.properties.NAME_JA = translationCache[name];
      appliedCount++;
    }
  }
  
  console.log(`  ✅ ${appliedCount}/${features.length}件に日本語名を適用`);
  return geojson;
}

/**
 * 翻訳キャッシュの読み込み/保存
 */
async function loadTranslationCache() {
  const cachePath = path.resolve(__dirname, '../public/data/translation-cache.json');
  try {
    const data = await fs.readFile(cachePath, 'utf-8');
    return JSON.parse(data);
  } catch (error) {
    return {};
  }
}

async function saveTranslationCache(cache) {
  const cachePath = path.resolve(__dirname, '../public/data/translation-cache.json');
  await fs.mkdir(path.dirname(cachePath), { recursive: true });
  await fs.writeFile(cachePath, JSON.stringify(cache, null, 2), 'utf-8');
}

/**
 * メイン処理
 */
async function main() {
  console.log('🌍 GeoJSONファイルに日本語名を追加する\n');

  let translationCache = await loadTranslationCache();
  console.log(`📦 翻訳キャッシュ: ${Object.keys(translationCache).length}件\n`);

  const files = [
    'world_bc2000.geojson',
    'world_bc500.geojson',
    // ... 他のファイル
  ];

  for (const filename of files) {
    console.log(`⏳ ${filename} を処理中...`);
    const url = `${HISTORICAL_BASE_URL}/${filename}`;
    
    const response = await fetch(url);
    const geojson = await response.json();
    
    const withJa = await addJapaneseNames(geojson, translationCache);
    
    // 保存
    const outputPath = path.resolve(__dirname, `../public/data/historical/${filename}`);
    await fs.mkdir(path.dirname(outputPath), { recursive: true });
    await fs.writeFile(outputPath, JSON.stringify(withJa, null, 2), 'utf-8');
  }

  await saveTranslationCache(translationCache);
  console.log(`\n✨ 完了! 翻訳キャッシュ: ${Object.keys(translationCache).length}件`);
}

main().catch(console.error);
```

### 使い方

```bash
# APIキーを設定
export GEMINI_API_KEY="your-api-key"

# スクリプト実行
node scripts/add-japanese-names.mjs
```

## 結果: 圧倒的な効率化

実際に実行してみた結果がこちら。

### 処理ログ

```
🌍 GeoJSONファイルに日本語名を追加する

📦 翻訳キャッシュ: 0件

⏳ world_bc2000.geojson を処理中...
  📦 バッチ 1/2 (100件)
  🤖 100件の国名をGemini Flash APIで翻訳中...
  📦 バッチ 2/2 (45件)
  🤖 45件の国名をGemini Flash APIで翻訳中...
  ✅ 145/145件に日本語名を適用

⏳ world_bc500.geojson を処理中...
  📦 バッチ 1/1 (74件)  // ← 新規74件のみ翻訳
  🤖 74件の国名をGemini Flash APIで翻訳中...
  ✅ 189/189件に日本語名を適用

⏳ world_bc323.geojson を処理中...
  📦 バッチ 1/1 (12件)  // ← さらに減少
  🤖 12件の国名をGemini Flash APIで翻訳中...
  ✅ 156/156件に日本語名を適用

... (以降はほぼキャッシュヒット)

✨ 完了! 翻訳キャッシュ: 847件
```

### 効率の比較

| 方式 | 翻訳回数 | API呼び出し | 処理時間 |
|------|---------|------------|----------|
| 愚直な方法 | 約3,000回 | 約30回 | 約10分 |
| **キャッシュ方式** | **約850回** | **約9回** | **約2分** |

**削減率: 約70%のトークン削減!**

## ポイント: なぜこんなに速いのか

### 1. 重複排除

```javascript
// 18ファイル中、"France"は何度も登場
// 愚直な方法: 18回翻訳
// キャッシュ方式: 1回だけ翻訳
```

### 2. バッチ処理

```javascript
// 100件ずつまとめて翻訳
const batch = uniqueNames.slice(i, i + 100);
const translations = await translateToJapanese(batch);
```

### 3. 永続化されたキャッシュ

```json
// public/data/translation-cache.json
{
  "France": "フランス",
  "Roman Empire": "ローマ帝国",
  "Mongol Empire": "モンゴル帝国",
  ...
}
```

次回実行時もこのキャッシュを再利用でき。

## アプリケーション側の実装

翻訳済みGeoJSONをローカルに配置:

```
public/
  data/
    modern/
      countries.geojson         # 日本語付き
    historical/
      world_bc2000.geojson      # 日本語付き
      world_bc500.geojson       # 日本語付き
      ...
```

検索実装:

```typescript
// src/components/SearchBar.tsx
const suggestions = useMemo(() => {
  const lowerQuery = query.toLowerCase();
  return countries.filter((country) => {
    const name = country.properties.NAME.toLowerCase();
    const nameJa = country.properties.NAME_JA?.toLowerCase() || "";
    
    // 英語・日本語両方で検索可能!
    return name.includes(lowerQuery) || nameJa.includes(lowerQuery);
  });
}, [query, countries]);
```

## まとめ

### この手法が有効なケース

- ✅ 大量の繰り返しデータの翻訳
- ✅ データ拡張・メタデータ付与
- ✅ 複数ファイルに同じエンティティが登場
- ✅ オフライン処理が可能

### キャッシュ方式のメリット

1. **トークン削減**: 重複翻訳を排除
2. **高速化**: 2回目以降はほぼキャッシュヒット
3. **コスト削減**: API呼び出し回数が激減
4. **再実行可能**: エラー時も途中から再開

### 応用例

この手法は翻訳以外にも使える:

- **カテゴリ分類**: LLMで一度分類したエンティティをキャッシュ
- **感情分析**: 同じテキストの重複分析を排除
- **要約生成**: 同じドキュメントの再要約を防止

## 参考リンク

- [Gemini API ドキュメント](https://ai.google.dev/gemini-api/docs)
- [Natural Earth データ](https://www.naturalearthdata.com/)
- [aourednik/historical-basemaps](https://github.com/aourednik/historical-basemaps)
