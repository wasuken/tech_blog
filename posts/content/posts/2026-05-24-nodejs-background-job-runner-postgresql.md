---
title: "Node.js でバックグラウンドジョブを自前実装する：PostgreSQL でジョブ管理"
date: 2026-05-24T00:00:00+09:00
draft: false
ai_assisted: true
tags: ["Node.js", "TypeScript", "PostgreSQL", "バックグラウンド処理", "個人開発"]
categories: ["個人開発"]
description: "BullMQ や外部キューを使わず、PostgreSQL の jobs テーブルと while(true) ループで軽量なジョブ管理システムを自前実装した話。Web UI からの操作・重複防止・ログ収集まで対応。"
---

BullMQ や外部キューサービスを使わずに、**PostgreSQL + while(true) ループ**でバックグラウンドジョブを管理する仕組みを作った。
「外部依存を増やしたくない」「DB を見るだけでジョブの状態がわかるようにしたい」という理由から。

---

## ⚠️ この実装の前提条件（割り切りポイント）

この仕組みは **「予算を抑えたい個人開発」かつ「アプリ単一インスタンス（1プロセス）」** での運用を前提に、あえてシンプルに作っています。
以下のトレードオフを理解した上で使ってください。

**1. 厳密な重複排除はしていない（レースコンディション）**
アプリ側で `isAnyRunning` をチェック → ジョブ作成という 2 ステップになっているため、ミリ秒単位で同時リクエストが来た場合はすり抜ける可能性があります。
厳密に防ぐなら、後述する DB 側の Partial Unique Index が必要です。

**2. マルチインスタンス非対応**
起動時にジョブを一括リセットしているため、コンテナを複数台並列で動かす（水平拡張する）場合は、他インスタンスで実行中のジョブを巻き添えにします。
複数台にするなら `worker_id` カラムを導入するか、一括リセットをやめてください。

「バグ」ではなく「この規模だからこその意図的な割り切り」です。BullMQ を検討する規模になったら移行サインと考えています。

---

## jobs テーブルの設計

```sql
CREATE TABLE jobs (
  id          SERIAL PRIMARY KEY,
  type        VARCHAR(50) NOT NULL,
  status      VARCHAR(20) NOT NULL DEFAULT 'pending',
  log         TEXT,
  started_at  TIMESTAMPTZ,
  finished_at TIMESTAMPTZ,
  created_at  TIMESTAMPTZ DEFAULT NOW()
);
```

`status` は `pending → running → done / error` と遷移する。
`log` カラムにジョブの実行ログを蓄積するので、Web UI から確認できる。

### （任意）重複をDB側で厳密に防ぐ Partial Unique Index

アプリ側チェックだけではレースコンディションが残ります。
より厳密に防ぎたい場合は、PostgreSQL の Partial Unique Index を張ります。

```sql
CREATE UNIQUE INDEX jobs_type_active_unique
  ON jobs (type)
  WHERE status IN ('pending', 'running');
```

これで `pending` または `running` の同じ `type` が 2 行以上存在できなくなります。
INSERT が競合した場合は PostgreSQL がエラーを返すので、アプリ側で `catch` して重複と判定できます。

> ただし今回の実装はシングルインスタンス前提のため、このインデックスは「念のため保険」程度の位置づけです。

---

## JobRunner の核心

```typescript
async createAndRun(type: JobType): Promise<number> {
  // 同じ type が既に running/pending なら弾く
  if (await this.jobRepo.isAnyRunning(type)) {
    throw new Error(`Job of type ${type} is already running.`)
  }

  const jobId = await this.jobRepo.create(type)

  // 非同期で実行（await しない）
  this.execute(jobId, type).catch((err) => {
    console.error(`Unhandled error in job ${jobId}:`, err)
  })

  return jobId  // すぐ jobId を返す
}
```

ポイントは `this.execute()` を **await しないこと**。
API リクエストは即座に `jobId` を返し、処理はバックグラウンドで走る。

---

## ログの収集

ジョブ実行中のログを DB にリアルタイムで書き込む。
`AsyncLocalStorage` を使って、UseCase 内の `logger.info()` が自動的にジョブのログとして記録される仕組み：

```typescript
const logContext = new AsyncLocalStorage<{ write: (msg: string) => void }>()

// ログ書き込み関数をコンテキストに注入
await logContext.run({ write: writeLog }, async () => {
  await this.analyzeUseCase.execute()
})

// logger 側では context から write を取り出して呼ぶ
export const logger = {
  info: (msg: string) => {
    const ctx = logContext.getStore()
    ctx?.write(`[INFO] ${msg}`)
    console.log(msg)
  }
}
```

UseCase 側は `logger.info()` を呼ぶだけで、ジョブコンテキストを意識しなくていい。

---

## auto-worker の while(true) ループ

```typescript
while (true) {
  const cycleSeconds = parseInt(
    await settingsRepo.get('cycle_interval_seconds') ?? '60', 10
  )

  await jobRunner.createAndRun('fetch-pdf').catch(() => {})
  await jobRunner.waitForJobType('fetch-pdf')

  await jobRunner.createAndRun('analyze').catch(() => {})
  await jobRunner.waitForJobType('analyze')

  await new Promise((r) => setTimeout(r, cycleSeconds * 1000))
}
```

`createAndRun` がエラー（重複など）を投げても `.catch(() => {})` で無視して続行。
サイクル間隔は DB の `settings` テーブルから読むので、UI から変更できる。

---

## プロセス再起動時のリセット

クラッシュや `docker restart` で `running` のまま止まったジョブが残る。
起動時にリセットする：

```typescript
await jobRepo.resetStaleRunningJobs()
```

```sql
UPDATE jobs
SET status = 'error',
    finished_at = NOW(),
    log = COALESCE(log, '') || E'\n[ERROR] Process was killed while running.'
WHERE status IN ('running', 'pending')
```

> **注意**: 冒頭の前提条件で述べた通り、これはシングルインスタンス前提の処理です。
> 複数台構成では他インスタンスの実行中ジョブを巻き添えにするため、そのまま使えません。

---

## まとめ

- `jobs` テーブルだけで状態管理・ログ収集・重複防止が完結する（**シングルインスタンス前提**）
- `createAndRun` は await しないことで非同期実行を実現
- `AsyncLocalStorage` でログを UseCase に透過的に流し込める
- 起動時の stale ジョブリセットを忘れずに
- スケールアウトが必要になったら BullMQ / pg-boss への移行を検討する
