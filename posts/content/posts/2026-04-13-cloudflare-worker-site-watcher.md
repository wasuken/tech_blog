---
title: "Cloudflare Workers でサイト監視 + Discord通知を作った"
date: 2026-04-13T12:00:00+09:00
draft: false
tags: ["Cloudflare", "Workers", "Discord", "監視", "インフラ"]
---

外形監視をどこに任せるか迷った。ラズパイでやるか、Cloudflareに任せるか。
外から見えるサービスの監視なら せっかくなので**Cloudflare Workers**にする。

自宅NWそこそこ落ちたりするので・・・。

## ラズパイと比較した

| | Cloudflare Workers | ラズパイ |
|---|---|---|
| 向いてる監視 | 外形監視（外からの死活確認） | 内部NW監視 |
| コスト | 無料 | 電気代のみ |
| メンテ | ほぼゼロ | たまに落ちる |
| ローカルNW確認 | ❌ | ✅ |
| 最小間隔 | 1分 | 自由 |

今回は `wasutech.dev` と `blog.wasutech.dev`、`techblog.wasutech.dev` の3つを監視したい。

## Cloudflare Workers 無料枠で十分な理由

[公式ドキュメント - Limits](https://developers.cloudflare.com/workers/platform/limits/) によると、無料プランは以下のとおり。

- Cron Triggers: 5個まで
- リクエスト: 1日10万回まで
- CPU時間: 10ms/invocation

今回のユースケースは「HTTPリクエスト投げてステータスコード確認して Discord Webhook 叩く」だけなので、CPU時間は 2〜3ms で収まる。5分間隔で3ドメイン監視しても 1日864リクエストなので余裕。

## コード全文

```typescript
// src/index.ts
const TARGETS = [
  { name: "wasutech.dev",          url: "https://wasutech.dev" },
  { name: "blog.wasutech.dev",     url: "https://blog.wasutech.dev" },
  { name: "techblog.wasutech.dev", url: "https://techblog.wasutech.dev" },
];

export default {
  async scheduled(_event: ScheduledEvent, env: Env, _ctx: ExecutionContext) {
    const results = await Promise.allSettled(
      TARGETS.map((t) => check(t.name, t.url))
    );

    const failures = results
      .map((r, i) => ({ result: r, target: TARGETS[i] }))
      .filter(({ result }) =>
        result.status === "rejected" ||
        (result.status === "fulfilled" && !result.value.ok)
      );

    if (failures.length > 0) {
      const lines = failures.map(({ result, target }) => {
        const detail =
          result.status === "rejected"
            ? (result.reason as Error).message
            : `HTTP ${(result.value as Response).status}`;
        return `🔴 ${target.name} | ${detail}`;
      });

      await notify(env.DISCORD_WEBHOOK, lines.join("\n"));
    }
  },
};

async function check(name: string, url: string): Promise<Response> {
  const res = await fetch(url, {
    method: "HEAD",
    signal: AbortSignal.timeout(10000),
  });
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res;
}

async function notify(webhookUrl: string, msg: string) {
  await fetch(webhookUrl, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ content: msg }),
  });
}

interface Env {
  DISCORD_WEBHOOK: string;
}
```

```toml
# wrangler.toml
name = "wasutech-monitor"
main = "src/index.ts"
compatibility_date = "2025-01-01"

[triggers]
crons = ["*/5 * * * *"]
```

### `Promise.allSettled` を使った理由

`Promise.all` だと1つが失敗した時点で残りを待たずに throw する。
`Promise.allSettled` なら3つ並列でチェックして、**全部の結果を集めてから**まとめて通知できる。
1メッセージにまとめることでDiscordがスパムにならない。

便利ー

## デプロイ手順

```bash
# 1. テンプレ作成（Hello World / TypeScript を選ぶ）
npm create cloudflare@latest wasutech-monitor
cd wasutech-monitor

# 2. src/index.ts を上のコードで上書き

# 3. wrangler.toml に crons を追記
# [triggers]
# crons = ["*/5 * * * *"]

# 4. Discord Webhook URLをシークレットとして登録
wrangler secret put DISCORD_WEBHOOK

# 5. デプロイ
wrangler deploy
```

`package.json` も `tsconfig.json` もテンプレのまま触らなくていい。
シークレットは `wrangler secret put` でベタ書きせずに管理する。

なお、私は

## 通知タイミング

異常時のみ通知。全ドメイン正常なら Discord は無音。

```typescript
if (failures.length > 0) {
  await notify(...)
}
// 正常時は何もしない
```

これだけで十分。監視が死んでるかどうかが不安なら、Cloudflare ダッシュボードの Workers > Cron Events から直近100件の実行履歴が確認できる。

## まとめ

外形監視なら Cloudflare Workers の無料枠で完結する。
サーバー管理不要・メンテ不要・コスト無料と三拍子揃っている。

内部NW監視（Proxmoxノードや k3s クラスタなど）は別途ラズパイで担当させると役割が明確になってよい。
