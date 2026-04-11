# デプロイメント (Deployment) / core specification

> 親仕様書: [deployment.md](deployment.md)
> 役割: core specification

## デプロイメント戦略概要

### デプロイ対象

| 対象 | プラットフォーム | 更新頻度 | 優先度 |
| ---- | ---------------- | -------- | ------ |
| Web フロントエンド | Cloudflare Pages | 機能追加・修正のたび | 高 |
| API バックエンド | Cloudflare Workers | 機能追加・修正のたび | 高 |
| データベース | Cloudflare D1 | スキーマ変更時 | 高 |

### デプロイフロー

1. 開発者がコードをプッシュ
2. GitHub Actions で CI（型チェック・Lint・テスト・ビルド）を実行
3. CI が成功したら main ブランチにマージ
4. Cloudflare Pages / Workers へ自動デプロイ
5. デプロイ完了後、ヘルスチェックで正常性を確認
6. 問題があれば Cloudflare ダッシュボードから即座にロールバック

### 品質ゲート

| チェック項目 | 基準 | 必須 |
| ------------ | ---- | ---- |
| TypeScript 型チェック | エラーゼロ | Yes |
| ESLint | エラーゼロ | Yes |
| ビルド | 成功 | Yes |
| ユニットテスト | カバレッジ 60% 以上 | Yes |
| E2E テスト | クリティカルパス通過 | No |

---

## Cloudflare デプロイ戦略

### プラットフォーム構成

| サービス | 用途 | 無料枠 |
| -------- | ---- | ------ |
| Cloudflare Pages | フロントエンド（Next.js） | 無制限リクエスト・500ビルド/月 |
| Cloudflare Workers | API バックエンド | 100,000 リクエスト/日 |
| Cloudflare D1 | SQLite データベース | 5GB・500万読み取り/日 |
| Cloudflare R2 | オブジェクトストレージ | 10GB・エグレス無料 |
| Cloudflare KV | セッション・キャッシュ | 100,000 読み/日 |

### カスタムドメイン設定

#### 設定手順

1. Cloudflare ダッシュボード → Pages → プロジェクト → Custom Domains
2. ドメインを追加（例: `senpai.yourdomain.com`）
3. DNS は Cloudflare 管理の場合は自動設定
4. SSL 証明書は自動発行（Cloudflare Universal SSL）
5. DNS プロパゲーションは即時（Cloudflare 管理ドメインの場合）

### 環境分離

#### 環境構成

| 環境 | 用途 | ブランチ | デプロイトリガー |
| ---- | ---- | -------- | ---------------- |
| 開発 | ローカル開発 | - | 手動（wrangler dev） |
| ステージング | 本番前検証 | `develop` | プッシュ時自動 |
| 本番 | ユーザー提供 | `main` | プッシュ時自動 |

#### 各環境で分離すべき項目

- データベースインスタンス（別々の D1 データベース）
- 環境変数・シークレット（wrangler pages secret）
- ログレベル（開発: debug、本番: warn）
- 機能フラグ（KV で管理）

#### Cloudflare Pages プレビュー

- PR ごとに自動でプレビュー環境を作成
- URL 形式: `<hash>.senpai-web.pages.dev`
- GitHub PR にデプロイ URL が自動コメント

---

## GitHub Actions CI/CD パイプライン

### ワークフロー構成

| ファイル | 用途 |
| -------- | ---- |
| `ci.yml` | PR 時の CI（Lint・型チェック・テスト・ビルド） |
| `web-cd.yml` | Web アプリ CD（Cloudflare Pages 自動デプロイ + Discord 通知） |

### CI ワークフロー要件（PR 時）

#### トリガー条件

- PR が main ブランチに対して作成されたとき
- PR に新しいコミットがプッシュされたとき

#### 実行ステップ

1. リポジトリコードの取得
2. pnpm のセットアップ（バージョン: 9.x）
3. Node.js のセットアップ（バージョン: 22.x LTS）
4. pnpm キャッシュの有効化
5. 依存関係のインストール（frozen-lockfile モード）
6. TypeScript 型チェックの実行
7. ESLint によるコード品質チェック
8. Next.js ビルドの確認
9. Vitest によるユニットテストの実行
10. カバレッジチェックと Codecov 連携（閾値 80% 未達で CI 失敗）

### CD ワークフロー要件（main マージ時）

#### 実行内容

1. Cloudflare Pages へ自動デプロイ（wrangler-action）
2. デプロイ完了後、Discord Webhook で通知を送信

---

## ロールバック戦略

### Cloudflare Pages でのロールバック

#### ロールバック手順

**ダッシュボードから**

1. Cloudflare ダッシュボード → Pages → Deployments
2. 前のデプロイメントを選択
3. 「Rollback to this deployment」をクリック

**CLI から**

```bash
wrangler pages deployment rollback <deployment-id> --project-name senpai-web
```

#### ロールバック判断基準

| 状況 | 対応 | 理由 |
| ---- | ---- | ---- |
| ビルド失敗 | 自動的に旧バージョン維持 | Cloudflare Pages の自動保護 |
| エラー率 > 5% | ロールバック検討 | 品質低下 |
| パフォーマンス劣化 > 30% | ロールバック検討 | ユーザー体験悪化 |
| マイナー不具合 | 次回修正で対応 | 影響範囲が限定的 |

### データベースマイグレーションのロールバック

#### 破壊的変更の回避原則

- カラム削除は即座に実行しない（非推奨マーク → 数リリース後に削除）
- テーブル削除は複数ステップで段階的に実施
- インデックス追加は本番適用前に影響をテスト

#### ロールバック戦略

| 変更種別 | 推奨アプローチ |
| -------- | -------------- |
| カラム追加 | NULL 許容またはデフォルト値を設定 |
| カラム名変更 | 新カラム追加 → データ移行 → 旧カラム削除 |
| 型変更 | 新カラム作成 → 段階的移行 |
| テーブル削除 | リネーム → 一定期間保持 → 完全削除 |

---

## 変更履歴

| 日付 | バージョン | 変更内容 |
| ---- | ---------- | -------- |
| 2026-04-09 | 2.0.0 | 旧デプロイ基盤・Electron 削除、Cloudflare（Pages/Workers/D1/R2/KV）へ移行 |
