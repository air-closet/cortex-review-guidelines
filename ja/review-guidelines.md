# cortex コードレビュー観点

このドキュメントは、cortex のコードレビューで最初に読む入口です。詳細な判断基準は [guidelines/](./guidelines/) 配下に分割し、自動レビュー agent も各ファイルを直接参照します。

## レビューの進め方

1. Product Graph MCP で変更対象の機能・依存・影響範囲を確認する。
2. 変更ファイルの種類に応じて、下記の詳細ガイドラインを読む。
3. 指摘の重要度は [guidelines/severity.md](./guidelines/severity.md) で分類する。
4. Critical / Major / Minor は具体的な修正理由と対象行を示す。Nit は任意対応として扱う。

## 必ず見る入口

| 観点 | 詳細 | 主な対象 |
|------|------|---------|
| アーキテクチャ | [guidelines/architecture.md](./guidelines/architecture.md) | `apps/`, `packages/`, 依存方向、共有パッケージ |
| Product Graph | [guidelines/graph-integrity.md](./guidelines/graph-integrity.md) | `@graph-*` タグ、ドキュメント整合性、Product Graph 例外 |
| セキュリティ | [guidelines/security.md](./guidelines/security.md) | 認証、入力検証、機密情報、外部 API |
| GCP SDK 利用 | [guidelines/gcp-sdk-usage.md](./guidelines/gcp-sdk-usage.md) | Cloud Run, GCS, Secret Manager, OTel |
| テスト・品質 | [guidelines/testing.md](./guidelines/testing.md) | テスト追加、境界条件、命名、コード品質 |
| 観測性・通知 | [guidelines/observability.md](./guidelines/observability.md) | ログ・Slack 通知・アラート本文を truncate しない、分割送信 |
| AI アンチパターン | [guidelines/ai-antipattern.md](./guidelines/ai-antipattern.md) | 幻覚 API、フォールバック濫用、デッドコード、スコープクリープ、不要な後方互換 |
| ドキュメント | [guidelines/document-writing.md](./guidelines/document-writing.md) | docs の配置、重複防止、生成物との整合性 |
| 影響範囲 | [guidelines/impact-analysis.md](./guidelines/impact-analysis.md) | 修正漏れ、呼び出し元、DB・API・画面への波及 |
| 再発防止 | [guidelines/recurrence-prevention.md](./guidelines/recurrence-prevention.md) | 障害 issue, lint 化, 横展開, guideline 追加 |
| 重要度分類 | [guidelines/severity.md](./guidelines/severity.md) | Critical / Major / Minor / Nit の判定 |

## 根本対応の原則

レビュー指摘は **該当 PR 内で根本原因に対処する**。症状緩和だけのパッチや回避策、別 PR への切り出しは原則として認めない。

- **根本対応必須**: 指摘されたアーキテクチャ違反 / 設計欠陥 / 仕様逸脱 / 安全性の欠陥は、その PR 内で原因そのものを直す。表層的な if 分岐の追加やエラー隠蔽だけで閉じない。これに反する対応は `REQUEST_CHANGES` で差し戻す。
- **後回しの禁止**: 「別 PR で対応」「次のセッションで対応」「スコープ外」「段階的に」を理由とした未対応・部分対応を禁止する。これらの理由でクローズしようとするコメントは reviewer 側が `REQUEST_CHANGES` で差し戻す。
- **TODO / FIXME による先送り禁止**: レビュー指摘の残課題を `TODO` / `FIXME` コメントとしてコードに残してマージしない。残課題を口頭・コメントで「あとで対応」と表明することも禁止。
- **降格禁止**: 後回し・段階的実装を理由として重要度を Nit に降格しない。詳細は [guidelines/severity.md](./guidelines/severity.md) の「重要度の降格禁止ルール」を参照する。

例外条件はこのドキュメントに型としては列挙しない。例外が必要な状況は都度判断し、合意内容を PR 上で明示する。

## 最重要ゲート

### Composable Architecture

- `packages/` は再利用可能な部品、`apps/` はそれらを組み合わせるサービスとして分離されているか。
- app 間の直接 import がなく、共有可能なロジックは適切な package に置かれているか。
- 新規 app / package は既存カテゴリの構造パターンと命名に従っているか。

詳細は [guidelines/architecture.md](./guidelines/architecture.md) を参照する。

### Product Graph 整合性

- 新規・変更された宣言に必要な `@graph-*` タグがあるか。
- コード、DB、docs、infra の接続が Product Graph 上で追えるか。
- 生成ドキュメントや索引に手動で重複した実装一覧を持ち込んでいないか。

詳細は [guidelines/graph-integrity.md](./guidelines/graph-integrity.md) と [guidelines/document-writing.md](./guidelines/document-writing.md) を参照する。

### 影響範囲分析

- 変更ノードから forward / backward の依存先・依存元を Product Graph で確認しているか。
- 修正漏れパターン（型変更の未伝播、`via:` フィールド不整合、`shares_topic` 片側変更、Repository → UseCase → Handler の連鎖未反映、テスト未追従、documented_by の乖離）が PR 内で潰されているか。
- セマンティック検索で発見された類似実装が同 PR で更新されているか。

詳細は [guidelines/impact-analysis.md](./guidelines/impact-analysis.md) を参照する。

### セキュリティと境界

- ユーザー入力、外部 API 応答、環境変数、Secret の扱いが境界で検証されているか。
- 必須値をフォールバックで隠さず、設定不整合を早期に失敗させているか。`baseUrl ?? ''`、`?: string` の optional 化、空文字・同一オリジン・`null` への暗黙フォールバックは禁止。必須設定は型で required にし、起動時 throw で固定する。
- 認証・認可・権限のチェックが呼び出し元任せになっていないか。

詳細は [guidelines/security.md](./guidelines/security.md) を参照する。

### GCP SDK 利用 (Cloud Run)

- Cloud Run + Node 24 + OTel で `@google-cloud/storage` / `@google-cloud/secret-manager` を新規 import していないか。
- GCS / Secret Manager は metadata server で OAuth token を取得し、REST API を `fetch` で直接呼び出しているか。
- `@google-cloud/bigquery` は禁止対象ではないが、GCP SDK を増やす場合は既存実績と OTel 影響を確認しているか。

詳細は [guidelines/gcp-sdk-usage.md](./guidelines/gcp-sdk-usage.md) を参照する。

### OTel 計測の有効化 (Cloud Run)

- 新規 Cloud Run Service / Job の Pulumi 定義 (`infra/.../index.ts`) で `OTEL_EXPORTER_OTLP_ENDPOINT` と `GRAFANA_CLOUD_API_KEY` を `valueSource.secretKeyRef` で envs に注入しているか。
- 上記 2 secret の `roles/secretmanager.secretAccessor` が該当 SA に `gcp.secretmanager.SecretIamMember` で bind されているか。
- アプリ側 (`apps/.../src/index.ts`) が `initOtel({ serviceName: '...' })` を呼んでいるか (env 不在だと `@cortex/otel` は init を skip し、trace / metrics / log が Grafana に送られず、例外も exception event 化されない)。

詳細は [guidelines/cloud-run-deploy.md](./guidelines/cloud-run-deploy.md) のパターン3を参照する。

### 公開 URL と DNS / Edge Router

- 新規または変更された `*.air-closet.ai` 公開サービスが、DNS レコードだけでなく実際の配信経路まで定義されているか。
- Cloud Run 系サービス（API / MCP / Bot / Pipeline）は `infra/dns/index.ts` のサブドメイン定義と `infra/dns/sandbox.ts` の `cloudRunRoutes` KV ルートが揃っているか。
- Cloudflare Pages 系 Web アプリは Pages project / custom domain / deploy 設定が揃い、Edge Router KV へ誤って登録していないか。
- フロントエンドと API を分離する場合、Web URL と API URL の両方について外形確認（例: `200`, `401 + X-Edge-Auth-Start`, `/health`）ができるか。

詳細は infra/edge-router.md _(cortex 内部参照)_ を参照する。

### 観測性・通知メッセージ

- ログ / Slack 通知 / アラート本文を `slice` / `…他 N 件` / 文字数上限で truncate していないか。
- 件数が多いケースは分割送信や構造化ログで全量を残す方針になっているか。
- 切らざるを得ない場合、切れた事実と参照先（Logs Explorer / GCS / BQ URL 等）を併記しているか。
- `catch` ブロックでエラーログを出さずに `null` / 空配列 / `undefined` / 静的フォールバック値を返してサイレントに握りつぶしていないか。再 throw する場合を除き `@cortex/otel/logger` の Pino logger で `{ err: serializeError(error), event, ...context }` を構造化ログとして出しているか。
- エラーレベル（fatal / error / warn）が型名（例: `NotFoundError`）ではなく、そのエラーが機能にとって持つ意味（**復旧要否**・**影響範囲**・**業務的に予見済みか**）で判定されているか。詳細な判定基準は [guidelines/observability.md](./guidelines/observability.md) の「ログレベルの判定基準」を参照する。

詳細は [guidelines/observability.md](./guidelines/observability.md) を参照する。

### テストと検証

- 変更された振る舞いに対するテストが先に追加されているか。
- config 契約、境界チェック、失敗系は再発防止テストで固定されているか。
- プロジェクト定義の `pnpm test`、`pnpm build`、`pnpm lint` などを優先して実行しているか。

詳細は [guidelines/testing.md](./guidelines/testing.md) を参照する。

### AI 生成コードのアンチパターン

- 幻覚 API・存在しないメソッド・誤った引数形式を呼び出していないか。
- 必須データに対する `??` / `||` / デフォルト引数フォールバック、`catch { return ''; }` 等で不確実性を隠していないか。
- リファクタ後に残った未使用関数・到達不能分岐・古い import・孤立した re-export がないか。
- 要求されていない機能追加・早すぎる抽象化・早すぎるキャッシュ戦略・不要な Legacy 互換マッピングを混入させていないか。
- 既存コードベースに存在するパターン（共通クライアント・命名規則・エラーハンドリング様式）から説明なく逸脱していないか。
- レビュー指摘に対し、根本修正ではなくテスト追加 / ドキュメント追加 / 無関係なリファクタで「対応したつもり」になっていないか。

詳細は [guidelines/ai-antipattern.md](./guidelines/ai-antipattern.md) を参照する。

### 再発防止

- 障害 / 不具合 issue の修正に lint 化 / 横展開 / 既存 guideline 追加 / 「何もしない」のいずれを選択したか PR description に明記されているか。
- 機械的に検出できる再発パターンを人力レビューだけに残していないか。
- 「とりあえずどこかに記録しておく」型のドキュメント追加（陳腐化前提のメモ集や TODO コメント）が含まれていないか。

詳細は [guidelines/recurrence-prevention.md](./guidelines/recurrence-prevention.md) を参照する。

### 品質基準の保護

- ガイドライン文書（`docs/guidelines/` 配下）・lint ルール（`packages/eslint-plugin-graph/` 等）・カバレッジ閾値（`vitest.config.ts`）など、品質基準を緩める変更が含まれていないか。
- 上記の緩和変更は **人間レビュアーの Approve を必須** とする。AI レビューはこれらを含む PR を Approve せず `REQUEST_CHANGES` で差し戻す。
- 「既存ルールが厳しすぎる」「現実に合わせる」等の理由による緩和も同様。基準の変更自体の妥当性は人間が判断する。
- 「既存コードがすでにこのパターンを使っている」「既存実装が違反しているので基準を合わせる」は緩和の正当化理由にならない。既存違反は別途修正すべき問題であり、基準を下げる根拠にはならない。

詳細は [guidelines/severity.md](./guidelines/severity.md) の「品質基準の緩和」を参照する。

## 判定フロー

| 判定 | 条件 | GitHub アクション | 効果 |
|------|------|------------------|------|
| Critical | セキュリティ、データ破壊、重大な本番障害、OOM リスク、ドキュメント不整合、`@graph-*` JSDoc 不一致・欠如、coverage 閾値引き下げ | `REQUEST_CHANGES` | マージブロック必須 |
| Major | 仕様逸脱、アーキテクチャ違反、テスト不足、運用不能、パフォーマンス重大問題 | `REQUEST_CHANGES` | マージブロック原則 |
| Minor | 保守性低下、命名改善、軽微なリファクタリング、局所的な不整合、将来の不具合リスク | `REQUEST_CHANGES` | resolved 必須（無視は不可） |
| Nit | スタイル好み、表記ゆれ、軽微な改善、任意の整理 | `APPROVE`（コメントで指摘） | ブロックしない |

重要度の詳細・降格禁止ルール・例外条件は [guidelines/severity.md](./guidelines/severity.md) を優先する。

## 自動レビュー運用

- 良い点だけのコメントは出さず、修正が必要な箇所に絞る。
- 指摘には対象ファイル、行番号、問題、修正理由、確認方法を含める。
- 同じ `family_tag` の指摘は重複させず、潜在箇所も同時に確認する。
- APPROVE する場合でも、残るリスクや未実行の検証があれば明記する。

## 関連ドキュメント

- DESIGN.md _(cortex 内部参照)_ - 設計方針と最新情報の読み始める入口
- VISION.md _(cortex 内部参照)_ - プロダクトビジョン
- [docs/guidelines/README.md](./guidelines/README.md) - 分割されたレビュー・ドキュメント基準
- [docs/guidelines/document-writing.md](./guidelines/document-writing.md) - ドキュメント作成・管理方針
- docs/infra/README.md _(cortex 内部参照)_ - インフラ構成と運用
- docs/product-graph/README.md _(cortex 内部参照)_ - Product Graph
- docs/code-graph/README.md _(cortex 内部参照)_ - Code Graph
