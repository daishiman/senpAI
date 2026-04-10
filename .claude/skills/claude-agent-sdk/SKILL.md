---
name: claude-agent-sdk
description: |
  Claude Agent SDK（@anthropic-ai/claude-agent-sdk）および直接Anthropic SDK（@anthropic-ai/sdk）を使用したエージェント統合の実装を専門とするスキル。
  query() API、Hooksシステム、Permission Control、ストリーミング処理、Direct SDKパターンを支援します。
  Webアプリ（Next.js / Cloudflare Workers）での統合に対応。

  Anchors:
  • Claude Agent SDK Official Docs / 適用: SDK API、Hooks、Permissions / 目的: 公式パターンに準拠した実装
  • Anthropic SDK (@anthropic-ai/sdk) / 適用: Direct SDK呼び出し / 目的: シンプルなサーバーサイド統合
  • TypeScript Handbook / 適用: 型定義、ジェネリクス / 目的: 型安全なSDK統合

  Trigger:
  Claude Agent SDKを使用したエージェント機能実装、query() APIストリーミング処理、Hooksシステム実装、Permission Control設計、MCP統合、Direct SDK統合を行う場合に使用。
  claude-agent-sdk, query API, PreToolUse, PostToolUse, PermissionRequest, MCP, ストリーミング, 権限制御, @anthropic-ai/sdk, Direct SDK, Cloudflare Workers, Next.js

allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Glob
  - Grep
---

# Claude Agent SDK

## 概要

Claude Agent SDK（`@anthropic-ai/claude-agent-sdk`）を使用したエージェント統合の実装を専門とするスキル。query() API、Hooksシステム、Permission Control、ストリーミング処理を支援します。

**対象環境**: Next.js (App Router / API Routes)、Cloudflare Workers

**対象言語**: TypeScript のみ

## 最新情報取得

SDK情報は頻繁に更新されるため、実装前に最新情報を確認してください。

```bash
# 最新情報を取得
node .claude/skills/claude-agent-sdk/scripts/fetch-latest-info.mjs

# npmパッケージ情報のみ
node .claude/skills/claude-agent-sdk/scripts/fetch-latest-info.mjs --category npm
```

詳細なURL一覧は `references/official-urls.md` を参照してください。

## ワークフロー

### Phase 1: 要件の明確化と設計方針の決定

**目的**: エージェント統合の要件を理解し、適切なパターンを選定する

**アクション**:

1. 使用するツール（Read, Edit, Bash等）を特定
2. Permission Control戦略を決定
3. `references/query-api.md` で基礎パターンを確認
4. `references/permission-control.md` で権限設計を確認

### Phase 2: SDK統合の実装

**目的**: query() APIとHooksを実装し、エージェント機能を構築する

**アクション**:

1. `assets/agent-handler-template.ts` を参照してAPIハンドラを実装
2. `references/hooks-system.md` でHooksパターンを確認
3. ストリーミング処理とエラーハンドリングを実装

### Phase 3: 検証と記録

**目的**: 成果物の品質を確認し、ナレッジを記録する

**アクション**:

1. `scripts/validate-agent-setup.mjs` で設定の検証
2. Permission Controlのテスト
3. 実装パターンのドキュメント化

## Task 仕様ナビ

| Task | 概要 | 対応する Phase | リソース |
| ---- | ---- | -------------- | -------- |
| query() API 基本実装 | ストリーミングメッセージ処理の基本 | Phase 1, 2 | query-api.md |
| Hooks 実装 | PreToolUse/PostToolUse/Permission | Phase 2 | hooks-system.md |
| Hooks Factory | createHooks, セキュリティチェック | Phase 2 | hooks-system.md |
| Permission Control 設計 | 権限ルールの設計と実装 | Phase 1, 2 | permission-control.md |
| エラーハンドリング | AbortSignal、タイムアウト、リトライ | Phase 2 | error-handling.md |
| リトライ機構 | Exponential Backoff, Jitter, エラー分類 | Phase 2 | retry-patterns.md |
| MCP 統合 | MCP サーバーとの連携 | Phase 2, 3 | mcp-integration.md |
| セキュリティ設計 | サンドボックス、権限制御 | Phase 2, 3 | security-sandboxing.md |
| Next.js API Route 統合 | App Router での Agent 統合 | Phase 2 | implementation-artifacts.md |
| Cloudflare Workers 統合 | Workers での Agent 統合 | Phase 2 | implementation-artifacts.md |

## パターン選択ガイド

### claude-agent-sdk vs 直接SDK使用

| 要件 | claude-agent-sdk | 直接SDK (`@anthropic-ai/sdk`) |
| ---- | ---------------- | ----------------------------- |
| Hooks (PreToolUse 等) | ✅ 必要 | ❌ 不要 |
| Permission Control | ✅ 必要 | ❌ 不要 |
| ストリーミング UI | ✅ 複雑 | ⚪ シンプル |
| API Route / Workers | ⚪ 可能 | ✅ 推奨 |
| バッチ処理 | ⚪ 可能 | ✅ 推奨 |

**推奨**:

- **対話型エージェント** → `@anthropic-ai/claude-agent-sdk`
- **バックグラウンド処理/バッチ** → `@anthropic-ai/sdk` 直接使用

### Direct Anthropic SDK Pattern（Next.js API Route）

Next.js API Route または Cloudflare Workers でシンプルなクエリを実行する場合のパターン。

```typescript
import Anthropic from "@anthropic-ai/sdk";

// APIキー管理（環境変数から取得）
function getApiKey(): string {
  const key = process.env.ANTHROPIC_API_KEY;
  if (!key) throw new Error("ANTHROPIC_API_KEY is not set");
  return key;
}

// Next.js API Route（App Router）
export async function POST(request: Request) {
  const { prompt, systemPrompt } = await request.json();

  const client = new Anthropic({ apiKey: getApiKey() });

  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), 30000);

  try {
    const response = await client.messages.create(
      {
        model: "claude-sonnet-4-6",
        max_tokens: 8192,
        ...(systemPrompt ? { system: systemPrompt } : {}),
        messages: [{ role: "user", content: prompt }],
      },
      { signal: controller.signal },
    );

    const textContent = response.content.find((b) => b.type === "text");
    const text = textContent?.type === "text" ? textContent.text : "";

    return Response.json({ result: text });
  } finally {
    clearTimeout(timeoutId);
  }
}
```

### SkillExecutor Pattern

フェーズベースのスキル実行パターン。進捗コールバック、キャンセル機能を含む。

```typescript
interface SkillExecutor {
  execute(
    phase: SkillPhase,
    context: ExecutionContext,
  ): Promise<SkillExecutionResult>;
  cancel(): void;
  onProgress(callback: (progress: number) => void): void;
  isExecuting(): boolean;
}

// スキルフェーズマッピング
const skillMap: Record<SkillPhase, string> = {
  hearing: "hearing-facilitator",
  structure: "structure-designer",
  html: "html-generator",
  modifier: "slide-modifier",
};
```

### AuthKeyService 統合パターン（Webアプリ）

Webアプリ環境でのセキュアな認証キー管理パターン。

```typescript
// 環境変数から取得（サーバーサイド専用）
function getAuthKey(): string {
  const key = process.env.ANTHROPIC_API_KEY;
  if (!key) throw new Error("API key not configured");
  return key;
}

// Cloudflare Workers での取得（Bindings使用）
interface Env {
  ANTHROPIC_API_KEY: string;
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const apiKey = env.ANTHROPIC_API_KEY;
    // ...
  },
};
```

**認証キー解決優先順位**:

1. `options.apiKey` で直接指定
2. 環境変数 `ANTHROPIC_API_KEY`（Next.js: `.env.local`）
3. Cloudflare Workers Bindings の `env.ANTHROPIC_API_KEY`

## ベストプラクティス

### すべきこと

- Permission Rules で適切な権限制御を設計する
- PreToolUse フックで危険なコマンドをブロックする
- AbortSignal を使用してタイムアウト処理を実装する
- ストリーミングメッセージを適切に UI 更新に反映する
- エラー発生時のフォールバック処理を実装する
- API キーはサーバーサイドのみで扱う（クライアントに露出させない）

### 避けるべきこと

- `permissionMode: 'bypassPermissions'` を本番環境で使用する
- Hook なしで危険なツール（Bash 等）を許可する
- ストリーミング中のエラーを無視する
- API キーをクライアントサイドのコードに含める
- `signal.aborted` のチェックを省略する

## クイックリファレンス

### パッケージインストール

```bash
pnpm add @anthropic-ai/claude-agent-sdk
```

### 基本使用例

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

const conversation = query({
  prompt: "Hello, Claude!",
  options: {
    tools: ["Read", "Edit"],
    permissionMode: "default",
    env: { ANTHROPIC_API_KEY: apiKey },
    abortController: new AbortController(),
  },
});

for await (const message of conversation) {
  console.log(message);
}
```

### Hook 実装例

```typescript
const options = {
  hooks: {
    PreToolUse: async (input, toolUseID, { signal }) => {
      if (input.toolName === "Bash" && input.args.command?.includes("rm -rf")) {
        return {
          proceed: false,
          message: "危険なコマンドは許可されていません",
        };
      }
      return { proceed: true };
    },
  },
};
```

## リソース参照

### 責務別ドキュメント

```bash
# query() API、SDKMessage型、ストリーミング
cat .claude/skills/claude-agent-sdk/references/query-api.md

# Hooksシステム（全イベント、実装パターン）
cat .claude/skills/claude-agent-sdk/references/hooks-system.md

# Permission Control（4層システム、ルール）
cat .claude/skills/claude-agent-sdk/references/permission-control.md

# エラーハンドリング（AbortSignal、タイムアウト）
cat .claude/skills/claude-agent-sdk/references/error-handling.md

# リトライパターン（Exponential Backoff, Jitter, エラー分類）
cat .claude/skills/claude-agent-sdk/references/retry-patterns.md

# MCP統合
cat .claude/skills/claude-agent-sdk/references/mcp-integration.md

# セキュリティとサンドボックス
cat .claude/skills/claude-agent-sdk/references/security-sandboxing.md

# 公式URL一覧
cat .claude/skills/claude-agent-sdk/references/official-urls.md

# タスク別実装成果物・ファイル一覧
cat .claude/skills/claude-agent-sdk/references/implementation-artifacts.md
```

### テンプレート参照

```bash
cat .claude/skills/claude-agent-sdk/assets/agent-handler-template.ts
cat .claude/skills/claude-agent-sdk/assets/use-agent-hook-template.ts
```

## 関連ドキュメント

| ドキュメント | パス | 説明 |
| ------------ | ---- | ---- |
| Agent SDK インターフェース仕様 | `.claude/skills/aiworkflow-requirements/references/interfaces-agent-sdk.md` | 統合システム設計仕様（型定義） |
| 実装成果物一覧 | `references/implementation-artifacts.md` | タスク別の成果物・実装ファイル |
