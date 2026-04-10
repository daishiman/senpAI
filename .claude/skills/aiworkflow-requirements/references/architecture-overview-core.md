# アーキテクチャ総論 / core specification

> 親仕様書: [architecture-overview.md](architecture-overview.md)
> 役割: core specification

## 概要

本ドキュメントはAIWorkflowOrchestratorプロジェクトで採用しているアーキテクチャ、設計原則、パターンの総論です。個別の詳細は各ドキュメントを参照してください。

---

## 設計思想

### 採用アーキテクチャ

| アーキテクチャ | 説明 | 適用範囲 |
|--------------|------|---------|
| **Layered Architecture** | 依存方向を外側→内側に統一した層構造 | モノレポ全体 |
| **Monorepo** | pnpm workspace による共通コード共有 | プロジェクト構造 |
| **Domain-Centric** | ドメイン層を依存ゼロで維持 | packages/shared/core/ |

**レイヤードアーキテクチャの採用理由:**

- Clean Architectureの完全な実装ではなく、個人開発規模に適した簡略化版
- 内側の層（core, types）は外部依存ゼロを維持し、テスタビリティを確保
- 複雑なDI（Dependency Injection）コンテナは使用せず、明示的な依存注入

### 設計原則

| 原則 | 説明 | 適用方法 |
|-----|------|---------|
| **単一責任の原則（SRP）** | 1つのモジュールは1つの責務のみ | サービス分割、Slice分割 |
| **依存性逆転の原則（DIP）** | 上位モジュールは下位に依存しない | インターフェース定義 |
| **関心の分離（SoC）** | UI/ロジック/データを分離 | レイヤー構成 |
| **最小権限の原則** | 必要最小限のアクセス権のみ付与 | Cloudflare API Token スコープ / Auth Middleware |

---

## レイヤー構成

### 依存方向（上位→下位）

| 順序 | レイヤー | コンポーネント | 依存先 |
|-----|---------|--------------|-------|
| 1 | Application Layer | apps/web/ (Next.js + Cloudflare Pages) | Integration Layer, Infrastructure Layer |
| 1 | Application Layer | apps/api/ (Cloudflare Workers) | Integration Layer, Infrastructure Layer |
| 2 | Integration Layer | packages/integrations/{service}/ (外部サービス連携ツール) | Domain Layer |
| 3 | Infrastructure Layer | packages/shared/infrastructure/ (DB, AI, 外部サービス) | Domain Layer |
| 4 | Domain Layer | shared/core/ (エンティティ) | なし |
| 4 | Domain Layer | shared/types/ (型定義, Zod) | なし |

**依存ルール**: 上位レイヤーは下位レイヤーにのみ依存可能。Domain Layerは外部依存ゼロを維持。

### レイヤー定義

| レイヤー | ディレクトリ | 責務 | 依存許可 |
|---------|------------|------|---------|
| ドメイン | packages/shared/core/ | エンティティ、インターフェース | なし |
| 型定義 | packages/shared/src/types/ | 型定義、Zodスキーマ | なし |
| ドメインサービス | packages/shared/src/services/ | ビジネスロジック | types/ のみ |
| インフラ | packages/shared/infrastructure/ | 外部サービス連携 | core/, types/ |
| インテグレーション | packages/integrations/{service}/ | 外部API連携ツールパッケージ | core/, types/ |
| UI | packages/shared/ui/ | 共通コンポーネント | core/ |
| Webアプリ | apps/web/ | Next.js App Router (Cloudflare Pages) | shared/*, integrations/* |
| APIサーバー | apps/api/ | Cloudflare Workers (Hono) | shared/*, integrations/* |

📖 詳細: [architecture-monorepo.md](./architecture-monorepo.md)

---

## デザインパターン

### 構造パターン

| パターン | 適用箇所 | 目的 | 参照 |
|---------|---------|-----|-----|
| **Facade** | SkillService, SkillCreatorService | 複雑なサブシステムへの単純なインターフェース | architecture-patterns.md |
| **Repository** | Cloudflare D1操作 | データアクセスの抽象化 | database-schema.md |
| **API Route Handler** | Hono ルート（apps/api/src/routes/） | HTTPリクエストルーティング | deployment-core.md |

### 振る舞いパターン

| パターン | 適用箇所 | 目的 | 参照 |
|---------|---------|-----|-----|
| **Result Type** | 全APIレスポンス | エラーハンドリングの統一 | error-handling.md |
| **Observer** | EventEmitter（Claude CLI） | ストリーミング出力 | arch-claude-cli.md |
| **State（Slice）** | Zustand状態管理 | UI状態の分離管理 | arch-state-management.md |

### アーキテクチャパターン

| パターン | 適用箇所 | 目的 | 参照 |
|---------|---------|-----|-----|
| **Slice Pattern** | Zustand Store | 機能単位の状態分離 | arch-state-management.md |
| **Integration Package** | packages/integrations/{service}/ | 外部サービス連携の再利用単位化 | arch-integration-packages.md |
| **Workflow Feature** | apps/web/features/{workflow}/ | インテグレーションを組み合わせたワークフロー | arch-integration-packages.md |
| **DB Graceful Degradation** | Cloudflare D1ハンドラー | D1障害時のフォールバック応答 | database-schema.md |
| **Auth Middleware** | Hono middleware | 認証・認可の一元管理 | security-principles.md |

📖 詳細: [architecture-patterns.md](./architecture-patterns.md)

---

## UI/UXアーキテクチャ

### Atomic Design

| 階層 | 説明 | 配置場所 |
|-----|------|---------|
| **Atoms** | 最小単位（Button, Input, Icon） | packages/shared/ui/atoms/ |
| **Molecules** | 機能単位（FormField, SearchBar） | packages/shared/ui/molecules/ |
| **Organisms** | セクション（Header, Sidebar） | packages/shared/ui/organisms/ |
| **Templates** | レイアウト構造 | 各アプリ内 components/templates/ |
| **Pages** | 具体的な画面 | 各アプリの app/ |

### プラットフォームガイドライン

| ガイドライン | 適用範囲 | 目的 |
|------------|---------|-----|
| **WCAG 2.1 AA** | 全UI | アクセシビリティ |
| **Responsive Design** | apps/web/ | モバイル・デスクトップ対応 |

### Web App レイアウト構成

| コンポーネント | 役割 | 実装場所 |
| --- | --- | --- |
| `AppLayout` | サイドバー / ヘッダー / メイン / モバイルナビの統合テンプレート | `apps/web/src/components/organisms/AppLayout/` |
| `GlobalNavStrip` | デスクトップ / タブレットのグローバルナビ | `apps/web/src/components/organisms/GlobalNavStrip/` |
| `MobileNavBar` | モバイル下部ナビゲーション | `apps/web/src/components/organisms/MobileNavBar/` |
| `useNavShortcuts` | キーボードショートカットと遷移 | `apps/web/src/hooks/useNavShortcuts.ts` |

TASK-SKILL-LIFECYCLE-01 以降、`SkillCenterView` は lifecycle の primary entry surface として扱い、create / use / improve の job guide を入口で提示する。

### Skill Advanced Views

| ビュー | 責務 | 配置先 |
|-------|------|-------|
| SkillChainBuilder | スキルチェーンパイプライン構築 | apps/web/features/skill-chain/ |
| ScheduleManager | スケジュール管理 | apps/web/features/schedule/ |
| DebugPanel | デバッグパネル | apps/web/features/debug/ |
| AnalyticsDashboard | 分析ダッシュボード | apps/web/features/analytics/ |

📖 詳細: [ui-ux-design-principles.md](./ui-ux-design-principles.md)

---

## セキュリティアーキテクチャ

### Cloudflare Workers セキュリティ設定

| 設定 | 値 / 方針 | 目的 |
|-----|----------|-----|
| CORS Policy | オリジン制限（Pages ドメインのみ許可） | クロスオリジンアクセス制御 |
| Rate Limiting | Cloudflare Rate Limiting ルール | API 乱用防止 |
| WAF | Cloudflare WAF（OWASP ルールセット） | 悪意あるリクエストのブロック |
| HTTPS 強制 | Cloudflare Always Use HTTPS | 通信の暗号化 |

### セキュリティパターン

| パターン | 説明 | 参照 |
|---------|-----|-----|
| **Zod Validation** | APIリクエスト境界での入力検証 | error-handling.md |
| **Auth Middleware** | Hono middleware で認証チェックを一元化 | security-principles.md |
| **Path Traversal Prevention** | パス検証（Unicode正規化含む） | security-principles.md |
| **CSP** | Content Security Policy（Next.js ヘッダー） | security-principles.md |
| **Secrets Isolation** | 本番/staging 別シークレット管理 | deployment-secrets-management.md |
| **Least Privilege** | Cloudflare API Token スコープ最小化 | deployment-secrets-management.md |

### 認証・認可

| 方式 | 用途 | 参照 |
|-----|-----|-----|
| OAuth 2.0 PKCE | ソーシャルログイン（Web） | security-principles.md |
| Supabase Auth | 認証プロバイダー | security-principles.md |
| JWT / Session Cookie | API 認証トークン | security-principles.md |

📖 詳細: [security-principles.md](./security-principles.md), [deployment-secrets-management.md](./deployment-secrets-management.md)

---

## 状態管理アーキテクチャ

### Zustand Slice構成

| Slice | 責務 | 実装ファイル |
|------|-----|------------|
| uiSlice | UI状態（currentView / responsiveMode / isNavExpanded / isMobileMoreOpen） | store/slices/uiSlice.ts |
| authSlice | 認証状態 | store/slices/authSlice.ts |
| chatSlice | チャット状態 | store/slices/chatSlice.ts |
| agentSlice | エージェント・スキル管理 | store/slices/agentSlice.ts |
| chatEditSlice | コード編集状態 | features/workspace-chat-edit/store/ |

### 状態管理原則

| 原則 | 説明 |
|-----|-----|
| **Single Source of Truth** | 状態は一箇所で管理 |
| **Immutable Updates** | 状態は不変更新 |
| **Slice Isolation** | 機能単位でSliceを分離 |
| **Type Safety** | StateCreator型による型安全性 |

📖 詳細: [arch-state-management.md](./arch-state-management.md)

### Conversation 永続化アーキテクチャ

| 項目 | 内容 |
|-----|------|
| データストア | Cloudflare D1（SQLite, WAL mode） |
| スキーマ | chat_sessions + chat_messages（4インデックス） |
| API エンドポイント | POST/GET /api/conversations, /api/conversations/:id/messages |
| 登録パターン | Hono ルート登録（apps/api/src/routes/conversations.ts） |
| 障害時動作 | Graceful Degradation（ERR_4006 DB_NOT_AVAILABLE） |

📖 詳細: [database-schema.md](./database-schema.md)

---

## データフローアーキテクチャ

### Web/Workers 通信フロー

| ステップ | レイヤー | コンポーネント | 処理内容 |
|---------|---------|--------------|---------|
| 1 | Browser | Next.js Components / Zustand Stores / Hooks | UIイベント発生 |
| 2 | Browser | fetch / SWR / React Query | HTTP リクエスト送信 |
| 3 | 境界 | HTTPS (CORS Policy) | オリジン検証 |
| 4 | Workers | Hono Router (apps/api/src/routes/) | リクエスト受信・ルーティング |
| 5 | Workers | Services (Facade) | ビジネスロジック実行 |
| 5 | Workers | Repositories (D1) | データ永続化 |
| 5 | Workers | packages/integrations/{service}/ | 外部サービス連携 |
| 6 | Workers → Browser | HTTP Response (JSON) | 結果をブラウザに返却 |

**セキュリティ**: 全通信はHTTPS + CORS Policy経由。Zod による入力バリデーション必須。

### Workers API ルート概要

| ルートグループ | パス | 責務 |
|-------------|------|-----|
| auth | /api/auth/* | 認証・認可（OAuth, JWT） |
| skills | /api/skills/* | スキル管理・実行 |
| skill-creator | /api/skill-creator/* | スキル作成・ワークフロー |
| conversations | /api/conversations/* | 会話セッション管理 |
| integrations | /api/integrations/* | 外部サービス連携ブリッジ |

### Skill Creator Runtime Orchestration Foundation

Skill Creator runtime を次の 3 層で分ける。

| 層 | コンポーネント | 役割 |
| --- | --- | --- |
| public bridge | `RuntimeSkillCreatorFacade` | `skill-creator:plan/execute-plan/improve-skill` の public response と degraded response を返す |
| workflow foundation | `ManifestLoader`, `WorkflowManifest*` | `workflow-manifest.json` の read / validate / normalize / cache と shared contract を担う |
| workflow state owner | `SkillCreatorWorkflowEngine` | `currentPhase` / `awaitingUserInput` / `verifyResult` / phase artifacts / `resumeTokenEnvelope` を保持し、source provenance snapshot を route snapshot と同一 envelope に固定する |

Task08 session persistence/resume contract では、この engine state を永続化の正本入力として扱う。`resumeTokenEnvelope` 自体は runtime snapshot、persisted checkpoint は別契約とし、checkpoint は phase boundary 単位。

`RuntimeSkillCreatorFacade` は state owner ではなく public bridge に留まり、`plan()` の review state 記録、`execute()` の `terminal_handoff` 早期 return、`integrated_api` 完了時の verify 遷移記録を engine へ委譲する。`success: false` と executor reject は verify pending に進めず review へ戻し、`verification_review` と失敗 snapshot を engine が保持する。

`skill-creator:execute-plan` は `{ accepted: true, planId }` の ack を返す fire-and-forget 入口として扱い、実行完了通知は `SKILL_CREATOR_WORKFLOW_STATE_CHANGED` snapshot relay に分離する。

📖 詳細: [architecture-patterns.md](./architecture-patterns.md)

---

## ディレクトリ構造

### モノレポ全体構成

| ディレクトリ | 役割 |
|------------|-----|
| apps/web/ | Next.js Webアプリ（Cloudflare Pages） |
| apps/web/app/ | App Router |
| apps/web/features/ | ワークフロー機能モジュール |
| apps/api/ | Cloudflare Workers バックエンド（Hono） |
| apps/api/src/routes/ | API ルート定義 |
| apps/api/src/services/ | Facade サービス層 |
| apps/api/src/adapters/ | LLM / 外部サービス Adapter |
| apps/api/wrangler.toml | Workers 環境設定 |
| packages/shared/ | 共有パッケージ |
| packages/shared/core/ | ドメイン層（依存ゼロ） |
| packages/shared/src/types/ | 型定義（依存ゼロ） |
| packages/shared/src/services/ | ドメインサービス |
| packages/shared/infrastructure/ | インフラ層 |
| packages/shared/ui/ | UIコンポーネント |
| packages/integrations/ | 外部サービス連携パッケージ群 |
| packages/integrations/{service}/ | 個別サービス連携ツール |
| docs/ | ドキュメント |
| .claude/skills/ | Claude Codeスキル |

### Web App 構造

| ディレクトリ | 役割 |
|------------|-----|
| apps/web/app/ | Next.js App Router（ページルーティング） |
| apps/web/src/components/ | UIコンポーネント（Atomic Design） |
| apps/web/src/components/atoms/ | 最小単位コンポーネント |
| apps/web/src/components/molecules/ | 機能単位コンポーネント |
| apps/web/src/components/organisms/ | セクションコンポーネント |
| apps/web/features/ | 機能モジュール（ワークフロー別） |
| apps/web/features/{feature}/components/ | 機能固有コンポーネント |
| apps/web/features/{feature}/hooks/ | 機能固有フック |
| apps/web/features/{feature}/store/ | 機能固有 Zustand Slice |
| apps/web/src/store/ | グローバル Store |
| apps/web/src/store/slices/ | Zustand Slice |
| apps/web/src/hooks/ | 共通フック |

### Workers API 構造

| ディレクトリ | 役割 |
|------------|-----|
| apps/api/src/index.ts | Workers エントリポイント |
| apps/api/src/routes/ | Hono ルートハンドラー |
| apps/api/src/services/ | Facade サービス（SkillService 等） |
| apps/api/src/services/skill/ | スキル管理サービス |
| apps/api/src/adapters/llm/ | LLM Adapter（Anthropic/OpenAI/Google） |
| apps/api/src/adapters/handoff/ | Handoff Adapter |
| apps/api/src/middleware/ | 認証・バリデーション Middleware |
| apps/api/src/db/ | Cloudflare D1 Repository 層 |

📖 詳細: [directory-structure.md](./directory-structure.md)

---

## データ構造（型システム）

### 型定義の配置

| カテゴリ | 配置場所 | 用途 |
|---------|---------|-----|
| 共通型 | `packages/shared/src/types/` | Web/Workers共通 |
| RAG型 | `packages/shared/src/types/rag/` | RAG機能 |
| スキル型 | `packages/shared/src/types/skill.ts` | スキル管理 |
| Agent SDK型 | `packages/shared/src/types/agent.ts` | Agent SDK連携 |
| API型 | `apps/api/src/types/` | Workers API リクエスト/レスポンス型 |

### 型定義原則

| 原則 | 説明 |
|-----|-----|
| **Zod First** | ランタイムバリデーション付き型定義 |
| **Infer Type** | `z.infer<typeof schema>` で型推論 |
| **Shared Types** | Web/Workers共通型は shared に配置 |
| **Result Pattern** | 全APIは `Result<T>` 型で統一 |

### 主要型定義ファイル

| ファイル | 内容 | 参照 |
|---------|-----|-----|
| interfaces-core.md | コア型定義 | interfaces-core.md |
| interfaces-rag.md | RAG型定義 | interfaces-rag.md |
| interfaces-agent-sdk.md | Agent SDK型定義 | interfaces-agent-sdk.md |
| interfaces-auth.md | 認証型定義 | interfaces-auth.md |

📖 詳細: [interfaces-core.md](./interfaces-core.md)

---
