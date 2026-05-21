# ガイドライン

cortex リポジトリのコードレビュー・ドキュメント作成の共通基準。

自動レビューシステム（`scripts/auto-review/`）の sub agent が各ファイルを直接参照する。

## ファイル一覧

| ファイル | 対象 | 自動レビュー agent |
|---------|------|-------------------|
| [architecture.md](./architecture.md) | アプリ種別パターン・共有パッケージ・データアーキテクチャ | `arch-reviewer` |
| [graph-integrity.md](./graph-integrity.md) | Product Graph 整合性（@graph-* タグ、ドキュメント整合性） | `graph-reviewer` |
| [security.md](./security.md) | セキュリティ（認証、入力検証、機密情報） | `security-reviewer` |
| [gcp-sdk-usage.md](./gcp-sdk-usage.md) | GCP SDK 利用制限（Cloud Run + OTel の Storage / Secret Manager 禁止） | `security-reviewer` |
| [testing.md](./testing.md) | テスト品質・コード品質・命名規則 | `test-reviewer` |
| [observability.md](./observability.md) | ログ・通知メッセージ作成ガイド（truncate 禁止・分割送信） | `test-reviewer` |
| [ai-antipattern.md](./ai-antipattern.md) | AI 生成コード特有のアンチパターン（幻覚 API / フォールバック濫用 / デッドコード / スコープクリープ等） | `ai-antipattern-reviewer` |
| [document-writing.md](./document-writing.md) | ドキュメントの書き方・管理方針 | `doc-reviewer` |
| [impact-analysis.md](./impact-analysis.md) | 影響範囲分析（Product Graph による修正漏れ検出） | `impact-reviewer` |
| [recurrence-prevention.md](./recurrence-prevention.md) | 障害 issue 修正の判定マトリクス（lint 化 / 横展開 / guideline 追加 / 何もしない） | orchestrator |
| [severity.md](./severity.md) | 重要度分類（Critical/Major/Minor/Nit）と判定基準 | orchestrator |
| [external-api-clients.md](./external-api-clients.md) | 外部 API クライアント共通パッケージの整備方針（OpenAPI spec 駆動） | - |
| [lint-rules.md](./lint-rules.md) | カスタム ESLint ルール作成・運用方針（`no-restricted-syntax` 直書き禁止） | - |
| [internal-member-identifier.md](./internal-member-identifier.md) | 社内メンバーを identifier として扱う際の共通ルール（email \| nickname 受付・Google Workspace 突合） | - |
| [cloud-run-deploy.md](./cloud-run-deploy.md) | Cloud Run デプロイの既知の罠（`.gcloudignore` 再インクルード・`pnpm deploy --legacy` 直接依存宣言・OTel exporter env 注入漏れ） | - |
| [package-publish.md](./package-publish.md) | `packages/*` を GitHub Packages (`@air-closet`) に public publish する共通ワークフロー・package.json 規約・main merge 自動 publish 運用 | - |
| [frontend.md](./frontend.md) | `apps/web/` フロントエンドの UI/UX レイアウト基準 | - |
