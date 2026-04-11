# インテグレーションパッケージアーキテクチャ

> 本ドキュメントは senpAI のシステム設計仕様書の一部です。
> 管理: .claude/skills/aiworkflow-requirements/
>
> **注意**: 本ドキュメントはパターン・管理体制の定義であり、コード例は理解を助けるための例示です。
> 具体的な機能（Google Meet 連携や Slack 通知など）を実装する場合は、本ドキュメントのパターンに従って
> 別途タスク仕様書を作成してください。

---

## 概要

senpAI は **ツールパッケージ** と **ワークフローパッケージ** の2層構造で外部連携を実現する。

| 層 | 場所 | 責務 | 例 |
| -- | ---- | ---- | -- |
| ツールパッケージ | `packages/integrations/` | 外部サービスとの接続（単一責務・再利用可能） | Slack 送信、Google Meet 動画取得 |
| ワークフローパッケージ | `apps/web/features/` | ツールを組み合わせた業務フロー | 議事録生成 → Slack 通知 |

---

## ディレクトリ構造

```
packages/
  integrations/                    # ツールパッケージ（再利用可能）
    slack/                         # Slack 連携
    google-meet/                   # Google Meet 連携
    google-drive/                  # Google Drive 連携
    google-calendar/               # Google Calendar 連携
    notion/                        # Notion 連携
    ...                            # 追加可能

apps/
  web/
    features/                      # ワークフローパッケージ（業務フロー）
      meeting-minutes/             # 例: Google Meet → 議事録 → Slack
      data-report/                 # 例: データ収集 → レポート生成
      ...                          # 追加可能
    app/
      api/
        integrations/
          [service]/
            webhook/route.ts       # Webhook 受信エンドポイント
```

---

## ツールパッケージ仕様

### 設計原則

| 原則 | 説明 |
| ---- | ---- |
| 単一責務 | 1 パッケージ = 1 外部サービス |
| 自己完結 | 依存は `packages/shared/core/` と外部 SDK のみ |
| 型安全 | 全 API の入出力を TypeScript 型 + Zod で定義 |
| テスト可能 | モック可能なインターフェース設計 |
| 再利用可能 | 複数ワークフローから呼び出し可能 |

### 標準ファイル構成

```
packages/integrations/{service}/
  index.ts          # 公開 API（外部から見えるのはここのみ）
  client.ts         # 外部 SDK ラッパー（認証・接続管理）
  types.ts          # TypeScript 型定義・Zod スキーマ
  webhook.ts        # Webhook 受信・検証（必要な場合）
  errors.ts         # エラー型定義
  package.json      # 独立した npm パッケージ
  README.md         # 使用方法
  __tests__/        # テスト
```

### 標準 client.ts パターン

```typescript
// packages/integrations/slack/client.ts
import { WebClient } from "@slack/web-api";
import type { SlackConfig, SlackMessage } from "./types";

export class SlackClient {
  private client: WebClient;

  constructor(config: SlackConfig) {
    this.client = new WebClient(config.botToken);
  }

  async sendMessage(channel: string, message: SlackMessage): Promise<void> {
    await this.client.chat.postMessage({
      channel,
      text: message.text,
      blocks: message.blocks,
    });
  }

  async uploadFile(channel: string, file: Buffer, filename: string): Promise<void> {
    await this.client.files.uploadV2({
      channel_id: channel,
      file,
      filename,
    });
  }
}
```

### 標準 index.ts パターン（公開 API）

```typescript
// packages/integrations/slack/index.ts
export { SlackClient } from "./client";
export { SlackWebhookHandler } from "./webhook";
export type { SlackConfig, SlackMessage, SlackChannel } from "./types";
export { slackConfigSchema, slackMessageSchema } from "./types";
```

---

## ワークフローパッケージ仕様

### 設計原則

| 原則 | 説明 |
| ---- | ---- |
| 業務フロー単位 | 1 パッケージ = 1 ユースケース（例: 議事録生成） |
| ツール組み合わせ | ツールパッケージを import して連携 |
| IWorkflowExecutor | 統一インターフェースを実装 |
| 独立性 | 他ワークフローパッケージに依存しない |

### 標準ファイル構成

```
apps/web/features/{workflow-name}/
  schema.ts         # 入出力 Zod スキーマ
  executor.ts       # IWorkflowExecutor 実装（フロー定義）
  executor.test.ts  # 統合テスト
  api.ts            # Next.js API Route / Cloudflare Workers ハンドラー
  components/       # UI コンポーネント（任意）
  hooks/            # カスタム React フック（任意）
```

### 標準 executor.ts パターン

```typescript
// apps/web/features/meeting-minutes/executor.ts
import { GoogleMeetClient } from "@repo/integrations-google-meet";
import { SlackClient } from "@repo/integrations-slack";
import { generateText } from "ai";
import { anthropic } from "@ai-sdk/anthropic";
import type { MeetingMinutesInput, MeetingMinutesOutput } from "./schema";

export class MeetingMinutesExecutor implements IWorkflowExecutor {
  async execute(input: MeetingMinutesInput): Promise<MeetingMinutesOutput> {
    // Step 1: Google Meet から動画/字幕を取得
    const meetClient = new GoogleMeetClient({ accessToken: input.googleToken });
    const transcript = await meetClient.getTranscript(input.meetingId);

    // Step 2: AI で議事録を生成
    const { text: minutes } = await generateText({
      model: anthropic("claude-sonnet-4-6"),
      prompt: `以下のトランスクリプトから議事録を生成してください:\n\n${transcript}`,
    });

    // Step 3: Slack に送信
    const slackClient = new SlackClient({ botToken: input.slackToken });
    await slackClient.sendMessage(input.slackChannel, {
      text: `📝 議事録が生成されました`,
      blocks: [{ type: "section", text: { type: "mrkdwn", text: minutes } }],
    });

    return { minutes, sentTo: input.slackChannel };
  }
}
```

---

## パターン例: 外部サービス連携ワークフロー（参考）

> **これは例示です。** このフローを実装するためのドキュメントではありません。
> 新しい外部連携を追加する際は、このパターンに倣って `packages/integrations/` と `apps/web/features/` に実装してください。

### 例: 動画処理 → AI生成 → 通知 フロー

### フロー

```
[Google Drive に動画保存]
        ↓ (Webhook または ポーリング)
[meeting-minutes ワークフロー起動]
        ↓
[Step 1] GoogleMeetClient.getTranscript()
        ↓  packages/integrations/google-meet
[Step 2] AI で議事録生成（Vercel AI SDK）
        ↓  packages/shared/infrastructure/ai
[Step 3] SlackClient.sendMessage()
        ↓  packages/integrations/slack
[Slack チャンネルに議事録投稿]
```

### Webhook 受信エンドポイント

```typescript
// apps/web/app/api/integrations/google-drive/webhook/route.ts
import { GoogleDriveWebhookHandler } from "@repo/integrations-google-drive";
import { MeetingMinutesExecutor } from "@/features/meeting-minutes/executor";

export async function POST(request: Request) {
  const handler = new GoogleDriveWebhookHandler({
    channelToken: process.env.GOOGLE_WEBHOOK_CHANNEL_TOKEN!,
  });

  const event = await handler.parseEvent(request);

  if (event.type === "VIDEO_UPLOADED") {
    const executor = new MeetingMinutesExecutor();
    await executor.execute({ fileId: event.fileId, ... });
  }

  return new Response("OK");
}
```

---

## ツールパッケージ一覧（計画）

| パッケージ名 | 外部サービス | 主な機能 | ステータス |
| ----------- | ------------ | -------- | ---------- |
| `@repo/integrations-slack` | Slack | メッセージ送信、ファイル共有、Bot | 計画中 |
| `@repo/integrations-google-meet` | Google Meet | 録画取得、字幕取得 | 計画中 |
| `@repo/integrations-google-drive` | Google Drive | ファイル監視、ダウンロード | 計画中 |
| `@repo/integrations-google-calendar` | Google Calendar | イベント取得・作成 | 計画中 |
| `@repo/integrations-notion` | Notion | ページ作成・更新 | 計画中 |
| `@repo/integrations-github` | GitHub | Issue・PR 操作 | 計画中 |
| `@repo/integrations-discord` | Discord | メッセージ送信、Webhook | 計画中 |

---

## 依存関係ルール

```
[ワークフローパッケージ]    apps/web/features/{name}/
    ↓ import
[ツールパッケージ]          packages/integrations/{service}/
    ↓ import
[共通ドメイン]              packages/shared/core/

禁止:
- ツールパッケージ → ワークフローパッケージ（逆方向）
- ワークフローパッケージ → 他ワークフローパッケージ（相互依存）
- ツールパッケージ → 他ツールパッケージ（相互依存）
```

---

## 新規ツールパッケージ追加手順

1. `packages/integrations/{service-name}/` を作成
2. `package.json` に `@repo/integrations-{service-name}` として登録
3. `pnpm-workspace.yaml` に `packages/integrations/*` が含まれていることを確認
4. `client.ts`, `types.ts`, `index.ts` を実装
5. `__tests__/` にユニットテストを追加
6. `apps/web/features/` の任意のワークフローで import して使用

---

## 新規ワークフローパッケージ追加手順

1. `apps/web/features/{workflow-name}/` を作成
2. `schema.ts` に Zod スキーマで入出力を定義
3. `executor.ts` に `IWorkflowExecutor` を実装
4. `executor.test.ts` に統合テストを追加
5. `api.ts` に Cloudflare Workers / Next.js API Route を追加
6. 必要に応じて `apps/web/app/api/integrations/` に Webhook エンドポイントを追加

---

## 変更履歴

| 日付 | バージョン | 変更内容 |
| ---- | ---------- | -------- |
| 2026-04-09 | 1.0.0 | 初版作成（パッケージベース外部連携アーキテクチャ） |
