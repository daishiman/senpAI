# シークレット・環境変数管理

> 本ドキュメントは senpAI のデプロイメント仕様書の一部です。
> 管理: .claude/skills/aiworkflow-requirements/

---

## 概要

senpAI は **Cloudflare** と **GitHub** の2箇所でシークレットを管理する。

| 管理場所 | 何を置くか | 理由 |
| -------- | ---------- | ---- |
| **Cloudflare** | ランタイムシークレット（外部 API キー、DB 接続情報） | Worker/Pages が直接使用するため |
| **GitHub Secrets** | デプロイシークレット（Cloudflare API Token）| CI/CD 自動化のため |
| **GitHub Variables** | 非機密設定値（ドメイン名、プロジェクト名） | 環境別の設定切り替えのため |

---

## 管理場所の判断フロー

```
APIキーはどこで使われるか？

Runtime（Workers/Pages で直接使用）
  → Cloudflare Secrets で管理

CI/CD（GitHub Actions で使用）
  → GitHub Secrets で管理

公開情報（ドメイン名など）
  → GitHub Variables または wrangler.toml [vars] で管理
```

---

## Cloudflare Secrets（ランタイム）

### Cloudflare Workers（バックエンド `apps/api/`）

```bash
# production 環境
wrangler secret put OPENAI_API_KEY --env production
wrangler secret put DATABASE_URL --env production
wrangler secret put SLACK_BOT_TOKEN --env production

# staging 環境
wrangler secret put OPENAI_API_KEY --env staging
wrangler secret put DATABASE_URL --env staging
wrangler secret put SLACK_BOT_TOKEN --env staging
```

| シークレット名 | 説明 | 環境 |
| -------------- | ---- | ---- |
| `OPENAI_API_KEY` | OpenAI API キー（AI機能） | production / staging |
| `ANTHROPIC_API_KEY` | Anthropic API キー（Claude） | production / staging |
| `DATABASE_URL` | Cloudflare D1 接続 URL | production / staging |
| `SLACK_BOT_TOKEN` | Slack Bot Token（通知機能） | production / staging |
| `DISCORD_WEBHOOK_URL` | Discord Webhook（内部通知） | production / staging |

### Cloudflare Pages（フロントエンド `apps/web/`）

```bash
# サーバーサイドのシークレット（NEXT_PUBLIC_ではないもの）
wrangler pages secret put API_SECRET_KEY --project-name=senpai-web
```

> **注意**: `NEXT_PUBLIC_*` の値はビルド時に埋め込まれるため、シークレットではなく Cloudflare Pages の「Environment Variables」で管理する。

| 変数名 | 説明 | 分類 |
| ------ | ---- | ---- |
| `NEXT_PUBLIC_API_URL` | API のベース URL（公開） | Environment Variable |
| `NEXT_PUBLIC_CLOUDFLARE_ACCOUNT_ID` | CF アカウント ID（公開） | Environment Variable |
| `API_SECRET_KEY` | サーバーサイドの秘密鍵 | Secret |

---

## GitHub Secrets（CI/CD 用）

GitHub リポジトリの `Settings > Secrets and variables > Actions` で管理。

### Required Secrets

| シークレット名 | 説明 | 使用箇所 |
| -------------- | ---- | -------- |
| `CLOUDFLARE_API_TOKEN` | Cloudflare API Token（デプロイ用） | web-cd.yml, backend-ci.yml |
| `CLOUDFLARE_ACCOUNT_ID` | Cloudflare アカウント ID | web-cd.yml, backend-ci.yml |
| `DISCORD_WEBHOOK_URL` | Discord Webhook（デプロイ通知） | web-cd.yml, backend-ci.yml |
| `CODECOV_TOKEN` | Codecov カバレッジアップロード | ci.yml |

### GitHub Variables（非機密設定値）

`Settings > Secrets and variables > Actions > Variables` で管理。

| 変数名 | 説明 | 例 |
| ------ | ---- | -- |
| `CLOUDFLARE_PAGES_PROJECT` | Cloudflare Pages プロジェクト名 | `senpai-web` |
| `CLOUDFLARE_WORKERS_DOMAIN` | Workers 本番ドメイン | `api.senpai.workers.dev` |
| `CLOUDFLARE_WORKERS_STAGING_DOMAIN` | Workers ステージングドメイン | `api-staging.senpai.workers.dev` |

---

## wrangler.toml の環境別設定

ランタイムで使用する**非機密**設定値（シークレットでないもの）は `wrangler.toml` の `[vars]` セクションで管理する。

```toml
# apps/api/wrangler.toml

name = "senpai-api"

[vars]
ENVIRONMENT = "production"
LOG_LEVEL = "warn"

[env.staging]
name = "senpai-api-staging"

[env.staging.vars]
ENVIRONMENT = "staging"
LOG_LEVEL = "debug"

[env.production]
name = "senpai-api"

[env.production.vars]
ENVIRONMENT = "production"
LOG_LEVEL = "warn"
```

> **シークレットは wrangler.toml に記載しない**。`wrangler secret put` で登録する。

---

## ローカル開発での設定

ローカルの環境変数は、**1Password Environments** を正本にする。
`gitignore` 付きの平文 `.env` を秘密の正本にしない。環境変数は vault item に押し込まず、Environment に `key=value` で持たせる。

### 推奨構成

1. `1Password Developer` を有効化する
2. `Developer > View Environments` でプロジェクト用 Environment を作る
3. 変数は `OPENAI_API_KEY=...` のように Environment に直接追加する
4. 必要なら `Local .env file` destination を `/Users/dm/secrets/.env` に mount する
5. リポジトリにはサンプルの `*.example` だけを置く

### 理由

- 1Password Environments はプロジェクトの環境変数を vault から分離して管理できる
- local `.env` file destination は plaintext をディスクに書かずに `.env` として読める
- `Secure Note` は一般メモ用途であり、環境変数の管理単位としては不適切
- `op run` を経由した secret reference ファイル管理より、Environment 正本の方が設定の意味が明確

### 例外的にファイルを使う場合

```bash
# 1Password Environments から mount される例
OPENAI_API_KEY=...
DATABASE_URL=...
SLACK_BOT_TOKEN=...
```

```bash
# /Users/dm/secrets/.env（1Password が mount、手編集しない）
NEXT_PUBLIC_API_URL=http://localhost:8787
API_SECRET_KEY=...
```

> ここでの `.env` は 1Password が mount する読み取り専用の destination であり、手で平文を置く場所ではない。

---

## セキュリティ原則

| 原則 | 説明 |
| ---- | ---- |
| シークレットの分離 | 本番と staging は別のシークレット値を使用する |
| 最小権限 | Cloudflare API Token は必要なスコープのみ付与 |
| ローテーション | 90日ごとにシークレットをローテーションする |
| 監査 | GitHub Actions のログにシークレット値が出力されないことを確認 |

---

## Cloudflare API Token の作成手順

```
Cloudflare Dashboard > My Profile > API Tokens > Create Token

テンプレート: Edit Cloudflare Workers
スコープ:
  - Account > Workers Scripts: Edit
  - Account > Cloudflare Pages: Edit
  - Account > D1: Edit（D1 を使用する場合）
  - Zone > Zone: Read（カスタムドメインを使用する場合）
```

---

## 変更履歴

| 日付 | バージョン | 変更内容 |
| ---- | ---------- | -------- |
| 2026-04-09 | 1.0.0 | 初版作成（Cloudflare/GitHub 2層シークレット管理） |
