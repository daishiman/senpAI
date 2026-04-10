# デプロイメント (Deployment)

## 概要
この親仕様書は分割後の entrypoint であり、詳細仕様と履歴は child companion へ分離した。

## 仕様書インデックス
| ファイル | 役割 | 主な見出し |
| --- | --- | --- |
| [deployment-core.md](deployment-core.md) | core specification | デプロイメント戦略概要 / Cloudflare デプロイ戦略 / GitHub Actions CI/CD パイプライン |
| [deployment-cloudflare.md](deployment-cloudflare.md) | Cloudflare仕様 | Cloudflare Pages/Workers/D1/R2/KV セットアップ・運用手順 |
| [deployment-branch-strategy.md](deployment-branch-strategy.md) | ブランチ戦略 | feature/dev/main 3層ブランチ・staging/production 環境分離 |
| [deployment-secrets-management.md](deployment-secrets-management.md) | シークレット管理 | Cloudflare/GitHub での API キー・シークレット管理方針 |
| [deployment-details.md](deployment-details.md) | detail specification | モニタリングとアラート / デプロイチェックリスト / GitHub Secrets の要件 |
| [deployment-history.md](deployment-history.md) | history bundle | 関連ドキュメント |

## 利用順序
- まずこの親仕様書で対象 child companion を選ぶ。
- 実装や契約の詳細は `core` / `details` / `advanced` 系を読む。
- 完了タスク、変更履歴、補助情報は `history` / `archive` 系を読む。

## 関連ドキュメント
- `indexes/quick-reference.md`
- `indexes/resource-map.md`
