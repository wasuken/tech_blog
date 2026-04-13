---
title: "CloudflareアラートをWorker + Gemini APIでDiscordに要約通知する"
date: 2026-04-12T21:00:00+09:00
draft: false
tags: ["Cloudflare", "Workers", "Gemini", "Discord", "JavaScript"]
---

Cloudflareの通知をDiscord Webhookに流していたら、通知が大量に届くようになってしまった。重要なアラートが埋もれるので、Cloudflare Workers + Gemini APIで要約してから投稿するようにした。

## 方針

Cloudflareの通知設定で追加できるアラートはとりあえず全部有効にしている。各アラートが実際に役立つかは様子を見ながら判断する予定。

ただし**セキュリティアラートはだいたい30分に数回届いた**ため、現時点では通知をスキップするようにした。それ以外のアラートは引き続き通知して様子見中。

## 構成

```
Cloudflare Alert
  → Worker受信
  → フィルタリング（スキップ対象なら早期リターン）
  → Gemini API で日本語要約
  → Discord Webhook に送信
```

## 前提

- Cloudflare WorkersにDiscord Webhook通知のWorkerが既にある
- Google AI StudioのAPIキーを持っている

## モデル選定

Geminiのモデルは料金帯がいくつかある。

```
gemini-2.5-pro > gemini-2.5-flash > gemini-2.5-flash-lite
```

Cloudflareアラートの要約程度であれば **gemini-2.5-flash-lite** で十分。一番安い。

最新のモデル名は公式ドキュメントで確認すること。
https://ai.google.dev/gemini-api/docs/models

## 実装

### フィルタリングの考え方

頻度の高いアラートはWorker側でスキップできるようにしている。`SKIP_ALERT_TYPES` に列挙したタイプが一致した場合、Gemini APIを呼ばずに早期リターンする。不要なAPI呼び出しも減るのでコスト面でも良い。

どのアラートがどの `alert_type` を持つかはCloudflareの公式ドキュメントで確認できる。
https://developers.cloudflare.com/notifications/notification-available/

### Worker コード

```javascript
// スキップしたいアラートタイプ（頻度が高くて邪魔なものを列挙）
const SKIP_ALERT_TYPES = [
  "security_alerts", // セキュリティアラート：30分に数回来るので無効化中
];

async function summarizeWithGemini(apiKey, input) {
  const prompt = `以下のCloudflareアラートを日本語で3行以内に要約してください。重要度（🔴高/🟡中/🟢低）も判定してください。\n\n${input}`;

  const res = await fetch(
    `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-lite-preview-06-17:generateContent?key=${apiKey}`,
    {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        contents: [{ parts: [{ text: prompt }] }]
      })
    }
  );

  const data = await res.json();
  console.log("Gemini response:", JSON.stringify(data));
  return data.candidates?.[0]?.content?.parts?.[0]?.text || "要約失敗";
}

export default {
  async fetch(request, env) {
    if (request.method !== "POST") {
      return new Response("ok");
    }

    let body;
    try {
      body = await request.json();
    } catch {
      return new Response("invalid json", { status: 400 });
    }

    // アラートタイプによるフィルタリング
    const alertType = body.text?.alert_type || body.alert_type || "";
    if (SKIP_ALERT_TYPES.some(t => alertType.includes(t))) {
      console.log(`Skipped alert type: ${alertType}`);
      return new Response("skipped");
    }

    // Geminiには常にbody全体を渡す
    const input = JSON.stringify(body).slice(0, 500);

    const summary = await summarizeWithGemini(env.GEMINI_API_KEY, input);

    const message = {
      username: "Cloudflare",
      embeds: [{
        title: body.text?.title || body.text?.description || "Cloudflare Notification",
        description: summary,
        color: 0xF6821F,
        timestamp: new Date().toISOString()
      }]
    };

    await fetch(env.DISCORD_WEBHOOK, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(message)
    });

    return new Response("ok");
  }
}
```

### シークレット登録

```bash
wrangler secret put GEMINI_API_KEY
wrangler secret put DISCORD_WEBHOOK
```

### 動作確認

```bash
curl -X POST https://<your-worker>.workers.dev \
  -H "Content-Type: application/json" \
  -d '{"text": {"title": "テスト通知", "description": "これはテストです"}}'
```

スキップ確認：

```bash
curl -X POST https://<your-worker>.workers.dev \
  -H "Content-Type: application/json" \
  -d '{"alert_type": "security_alerts", "text": {"title": "セキュリティテスト"}}'
# → "skipped" が返ってくればOK
```

## ログ確認

Workerのログは Cloudflare Dashboard → Workers & Pages → 該当Worker → **Observability** で確認できる。

ローカルでリアルタイム確認したい場合は：

```bash
wrangler tail
```

## ハマったポイント

### spending cap に引っかかる

Gemini APIで `RESOURCE_EXHAUSTED` エラーが返ってくる場合、APIキーの問題ではなくAI Studio側のspending capに達している可能性がある。

```json
{
  "error": {
    "code": 429,
    "message": "Your project has exceeded its monthly spending cap.",
    "status": "RESOURCE_EXHAUSTED"
  }
}
```

https://aistudio.google.com/billing でspending capを確認・変更する。

別プロジェクトで上限に達したAPIキーを使い回していると気づきにくいので注意。

## まとめ

- Cloudflareの通知はとりあえず全部有効にして様子見する運用にしている
- 頻度が高くて邪魔なアラートは `SKIP_ALERT_TYPES` に追加してWorker側でフィルタリング
- セキュリティアラートは30分に数回来るので現時点ではスキップ中
- フィルタリングで早期リターンするのでGemini APIの無駄な呼び出しも減る
- モデルはflash-liteで十分、コスト低い
- spending capはAI Studio側で管理されているので別プロジェクトの使用量も把握しておく
