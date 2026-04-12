---
title: "Cloudflareの全通知をDiscord Webhookに飛ばす"
date: 2026-04-12T18:00:00+09:00
draft: false
tags: ["Cloudflare", "Discord", "Workers", "Bash", "自動化"]
categories: ["インフラ"]
---

CloudflareのNotificationsはUIから1個ずつ設定するのがとにかくだるい。
種別も多いし、手動でDiscord Webhookを登録していくのは非現実的。

Cloudflare APIとWorkersを組み合わせて全通知を一括登録する。

## 構成

```
Cloudflare Notifications
        ↓
  Cloudflare Worker (受け口)
        ↓
  Discord Webhook
```

Cloudflare NotificationsはWebhook送信に対応しているので、Workerを受け口にしてDiscordに転送する。

## 1. Discord Webhookを作成

通知を飛ばしたいDiscordチャンネルの設定から作成する。

**チャンネル設定 → Integrations → Webhooks → New Webhook → Copy Webhook URL**

## 2. Cloudflare Workerを作成

Cloudflareダッシュボードから **Workers & Pages → Create → Hello World** で作成する。

名前は `cf-notify-discord` など適当につけてDeploy。

エディタ画面で以下のコードに置き換える。

```javascript
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

    const message = {
      username: "Cloudflare",
      embeds: [{
        title: body.text?.title || "Cloudflare Notification",
        description: body.text?.description || JSON.stringify(body, null, 2),
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

Discord WebhookのURLはWorkerの**Settings → Variables and Secrets**でシークレットとして登録する。

- 変数名: `DISCORD_WEBHOOK`
- 値: DiscordのWebhook URL

## 3. API Tokenを作成

**My Profile → API Tokens → Create Token → Create Custom Token**

必要な権限は以下のみ。

| スコープ | リソース | 権限 |
|---|---|---|
| Account | Notifications | Edit |

## 4. 全通知種別を一括登録するスクリプト

```bash
#!/bin/bash

ACCOUNT_ID="${1}"
API_TOKEN="${2}"
WEBHOOK_URL="${3}"  # WorkerのURL

if [ -z "$ACCOUNT_ID" ] || [ -z "$API_TOKEN" ] || [ -z "$WEBHOOK_URL" ]; then
  echo "Usage: $0 <account_id> <api_token> <worker_url>"
  exit 1
fi

BASE_URL="https://api.cloudflare.com/client/v4/accounts/${ACCOUNT_ID}/alerting/v3"
AUTH_HEADER="Authorization: Bearer ${API_TOKEN}"

echo "=== 利用可能なアラート種別を取得 ==="
ALERTS=$(curl -s -X GET "${BASE_URL}/available_alerts" \
  -H "${AUTH_HEADER}" \
  -H "Content-Type: application/json")

ALERT_TYPES=$(echo "$ALERTS" | jq -r '.result | to_entries[].value[].type')

echo "=== Webhook登録 ==="
WEBHOOK_RESP=$(curl -s -X POST "${BASE_URL}/destinations/webhooks" \
  -H "${AUTH_HEADER}" \
  -H "Content-Type: application/json" \
  --data "{
    \"name\": \"Discord Worker\",
    \"url\": \"${WEBHOOK_URL}\"
  }")

WEBHOOK_ID=$(echo "$WEBHOOK_RESP" | jq -r '.result.id')
echo "Webhook ID: ${WEBHOOK_ID}"

if [ -z "$WEBHOOK_ID" ] || [ "$WEBHOOK_ID" = "null" ]; then
  echo "Webhook登録失敗"
  echo "$WEBHOOK_RESP"
  exit 1
fi

echo "=== 全アラート種別にNotification登録 ==="
for ALERT_TYPE in $ALERT_TYPES; do
  echo -n "登録中: ${ALERT_TYPE} ... "
  RESP=$(curl -s -X POST "${BASE_URL}/policies" \
    -H "${AUTH_HEADER}" \
    -H "Content-Type: application/json" \
    --data "{
      \"name\": \"${ALERT_TYPE}\",
      \"enabled\": true,
      \"alert_type\": \"${ALERT_TYPE}\",
      \"mechanisms\": {
        \"webhooks\": [{\"id\": \"${WEBHOOK_ID}\"}]
      }
    }")

  SUCCESS=$(echo "$RESP" | jq -r '.success')
  if [ "$SUCCESS" = "true" ]; then
    echo "OK"
  else
    ERROR_CODE=$(echo "$RESP" | jq -r '.errors[0].code')
    if [ "$ERROR_CODE" = "17103" ]; then
      echo "SKIP (フィルター必須)"
    else
      echo "FAIL"
      echo "$RESP" | jq '.errors'
    fi
  fi
done

echo "=== 完了 ==="
```

実行方法。

```bash
chmod +x setup_cf_notifications.sh
./setup_cf_notifications.sh <account_id> <api_token> <worker_url>
```

`jq` が必要なので未インストールの場合は入れておく。

```bash
# Arch Linux
sudo pacman -S jq

# Ubuntu/Debian
sudo apt install jq
```

### Account IDの確認

Cloudflareダッシュボードのトップページ右サイドバーに表示されている32文字の英数字。  
`cfut_`から始まる文字列はAPI Tokenなので間違えないこと。

## 5. 動作確認

WorkerのURLに直接POSTしてDiscordに通知が飛ぶか確認する。

```bash
curl -X POST "https://cf-notify-discord.{サブドメイン}.workers.dev" \
  -H "Content-Type: application/json" \
  --data '{
    "text": {
      "title": "テスト通知",
      "description": "Cloudflare通知のテストです"
    }
  }'
```

Discordに飛んでくれば完了。

## 注意点

- エラーコード `17103` が出る種別はフィルターの指定が必須なためSKIPされる。これらは手動で個別設定が必要
- API Tokenは作業後に削除しておく
- Cloudflare Workers無料枠は1日10万リクエストなので通知の受け口程度であれば実質無料で運用できる

## 参考

- [Cloudflare Notifications API](https://developers.cloudflare.com/api/resources/alerting/subresources/policies/methods/create/)
- [Cloudflare Workers](https://developers.cloudflare.com/workers/)
- [Discord Webhooks](https://docs.discord.com/developers/resources/webhook)
