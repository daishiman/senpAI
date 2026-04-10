# ブランチ戦略・環境構成

> 本ドキュメントは senpAI のデプロイメント仕様書の一部です。
> 管理: .claude/skills/aiworkflow-requirements/

---

## ブランチ戦略

senpAI は **3層ブランチ構成** で開発・ステージング・本番環境を分離する。

```
feature/xxx  →  dev  →  main
（機能開発）    （開発環境）  （本番環境）
```

| ブランチ | 目的 | デプロイ先 | 保護ルール |
| -------- | ---- | ---------- | ---------- |
| `feature/*` | 機能単位の開発 | なし（ローカルのみ） | 直接 push 禁止 |
| `dev` | 統合・開発環境 | Cloudflare (staging) | PR レビュー 1 名必須 |
| `main` | 本番環境 | Cloudflare (production) | PR レビュー 2 名必須、force push 禁止 |

---

## フロー

```
1. feature/* ブランチで機能開発
2. feature/* → dev へ PR & マージ（1名レビュー）
   → staging 環境へ自動デプロイ
   → 動作確認・QA
3. dev → main へ PR & マージ（2名レビュー + QA 完了）
   → production 環境へ自動デプロイ
```

---

## 環境マッピング

| 環境 | GitHub ブランチ | Cloudflare 環境名 | 用途 |
| ---- | --------------- | ----------------- | ---- |
| ローカル | `feature/*` | N/A | 機能開発・ユニットテスト |
| ステージング | `dev` | `staging` | 統合確認・QA・デモ |
| 本番 | `main` | `production` | エンドユーザー向け |

---

## CI/CD トリガー対応表

| ワークフロー | ブランチ | 動作 |
| ------------ | -------- | ---- |
| `ci.yml` | `main`, `dev`, PR to `main`/`dev` | Lint・Typecheck・Test・Build |
| `web-cd.yml` | `main` push | Cloudflare Pages production デプロイ |
| `web-cd.yml` | `dev` push | Cloudflare Pages staging デプロイ |
| `backend-ci.yml` | `main` push | Cloudflare Workers production デプロイ |
| `backend-ci.yml` | `dev` push | Cloudflare Workers staging デプロイ |

---

## GitHub 環境保護ルール（推奨設定）

### `production` 環境

```
Settings > Environments > production:
- Required reviewers: 2名
- Wait timer: 0 分
- Deployment branches: main のみ
- Environment secrets: （本番用シークレット）
```

### `staging` 環境

```
Settings > Environments > staging:
- Required reviewers: 0名（自動デプロイ）
- Deployment branches: dev のみ
- Environment secrets: （staging 用シークレット）
```

---

## ブランチ保護ルール（推奨設定）

### `main` ブランチ

```
Settings > Branches > main:
- Require pull request reviews: 2
- Require status checks to pass: ci / Validate Build
- Require branches to be up to date before merging: ON
- Restrict who can push: admins only
- Allow force pushes: OFF
- Allow deletions: OFF
```

### `dev` ブランチ

```
Settings > Branches > dev:
- Require pull request reviews: 1
- Require status checks to pass: ci / Validate Build
- Allow force pushes: OFF
```

---

## 変更履歴

| 日付 | バージョン | 変更内容 |
| ---- | ---------- | -------- |
| 2026-04-09 | 1.0.0 | 初版作成（feature/dev/main 3層ブランチ戦略） |
