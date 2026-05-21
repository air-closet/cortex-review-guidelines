# アーキテクチャ・設計パターン

## 根本思想：Composable Architecture

cortex の設計思想は **「小さな機能単位を設計し、それらを組み合わせてサービスを構成する」** Composable である。

- `packages/` に**再利用可能な部品**を配置する（ドメインロジック、インフラ共通、型定義、技術基盤）
- `apps/` の各サービスは `packages/` の部品を組み合わせて構成する
- 新しいサービスを作る際は、まず既存の部品で組み立てられないかを検討し、足りない部品だけを新たに設計する

各項目に **severity** と **scope** を明記する。auto-reviewer は scope に該当しない PR では当該項目を発火させない。

| severity | scope | 観点 |
|---|---|---|
| Major | 全 app | apps 間の直接 import（`../../apps/other/`）が存在しないか（必ず `packages/` 経由） |
| Major | 全 app | 新しいロジックが適切な粒度の「部品」として設計されているか（1 ファイル 1 責務） |
| Major | 全 app | アプリ固有ロジックと共有可能ロジックが分離されているか |
| Major | 全 app | 複数 app が同じ writer/reader 契約（BQ カラム値、Firestore key、Pub/Sub payload、共有 enum 等）を参照する場合、契約値そのもの（リテラル定数・型）を `packages/` に切り出して両 app から import しているか[^writer-reader-sot] |
| Minor | 全 app | 既存の `packages/` の部品を再利用しているか（車輪の再発明をしていないか） |
| Minor | 全 app | `packages/` への切り出し判断は **rule of three**（実際に 2 つ以上の app で重複が発生した時点で `packages/` へ抽出する）。「将来使うかも」を理由とした投機的切り出しは利用者 0 のパッケージを生む YAGNI 違反として禁止 |

[^writer-reader-sot]: 実例: styling-pattern-watch の transformer が BQ に書く `segment_name` のデフォルト値（`'全体（在庫あり）'`）と api がダッシュボード描画用に WHERE 句で使う値（`'all'`）が両 app に独立して書かれていたため、片方しか変わっていない状態でデプロイされ、API レスポンスが空配列になりダッシュボードが全 0 表示になった。修正で `@cortex/styling-pattern-watch-core` を作成し、`defaultSegments()` を SoT として両 app から import するようにした。AST レベルでの一般的な lint 検出は難しい（任意の文字列リテラルが対象になりうる）ため、本ガイドラインのチェック項目で扱う。

## アプリタイプ別の構造パターン

`apps/` 配下にはカテゴリ別のサブディレクトリがあり、各カテゴリには固有の構造パターンが確立されている。新しいアプリを追加する際は、**同カテゴリの既存アプリを参考にして同じパターンに従う**こと。

### API（`apps/api/`）

Feature-Based DDD + Clean Architecture。`features/{name}/` の中に `api/`, `domain/`, `application/`, `infrastructure/` のレイヤーを持つ。

| severity | scope | 観点 |
|---|---|---|
| Major | `apps/api` | 依存方向 `API → Application → Domain ← Infrastructure` を守っているか |
| Major | `apps/api` | Use Case が単一責任か（1 ファイル = 1 ビジネスフロー） |
| Major | `apps/api` | OpenAPI スキーマに `.openapi()` でドキュメントが付与されているか |

### Bot（`apps/bot/`）

Slack Bot。**Bot 自身はビジネスロジックを持たない**。Slack イベントの受信・ルーティング・応答のみを担当。

| severity | scope | 観点 |
|---|---|---|
| Major | `apps/bot` | Bot がビジネスロジックを直接実装していないか（`packages/domain-*` に委譲すべき） |
| Major | `apps/bot` | handlers が薄く、ロジックが features に分離されているか |

### Pipeline（`apps/pipeline/`）

外部サービスからのデータ収集。フラットな構成。

| severity | scope | 観点 |
|---|---|---|
| Critical | `apps/pipeline` | Pub/Sub リトライでの冪等性が確保されているか |
| Major | `apps/pipeline` | ストリーミング転送を使用しているか（メモリに全データ載せない） |

### CLI（`apps/cli/`）

ローカル実行専用の workspace package。`pnpm exec <bin-name>` で呼び出す CLI 群を bucket 単位でまとめる。

| severity | scope | 観点 |
|---|---|---|
| Critical | `apps/cli` | Product Graph 例外運用の詳細が [graph-integrity.md](./graph-integrity.md) と [../review-guidelines.md](../review-guidelines.md) に反映され、CLI 向け docs と矛盾していないか |
| Major | `apps/cli` | デプロイ対象の app ではなく、ローカル運用ツールとして設計されているか |
| Major | `apps/cli` | `package.json` / `tsconfig.json` / `oxlint.config.ts` / `vitest.config.ts` を持つ独立 package になっているか |
| Minor | `apps/cli` | 外部呼び出し面が `bin` に集約され、`src/*.ts` 直叩き前提になっていないか |

### Generator（`apps/generator/`）

AI 分析・生成バッチ（KPI、Bug Evaluation、Stylings Analysis 等）。フラットな src/ で `bigquery-client.ts` / `firestore-client.ts` / `config.ts` / 業務ロジック（例: `kpi-calculator.ts`）を並べる。実行頻度は Cloud Scheduler 駆動が中心。**Reference: `apps/generator/kpi/`**。

| severity | scope | 観点 |
|---|---|---|
| Major | `apps/generator` | LLM / BQ / Firestore クライアントが個別ファイルに分離され、テストで mock しやすい構造になっているか |
| Major | `apps/generator` | バッチの再実行時に重複生成・上書き破壊が起きないか（冪等性） |
| Minor | `apps/generator` | 業務ロジックを純粋関数に切り出してユニットテスト可能になっているか |

### Transformer（`apps/transformer/`）

ETL / Embedding / BQ ロード（biz-graph、image-processor、styling-pattern-watch 等）。`build-*.ts` / `generate-embeddings.ts` のような phase ファイルで責務分割する。**Reference: `apps/transformer/biz-graph/`**。

| severity | scope | 観点 |
|---|---|---|
| Major | `apps/transformer` | phase（build / embed / similarity 等）が個別ファイルに分離されているか（1 ファイル 1 phase） |
| Major | `apps/transformer` | 中間データを stream で扱っているか（メモリに全件展開しない） |
| Minor | `apps/transformer` | 部分再実行（特定 phase だけ）が可能な構造になっているか |

### MCP（`apps/mcp/`）

MCP サーバー（cortex-product-graph-server、code-graph-server 等）。`server.ts` + `tool-registry.ts` + `mcp/` の構成が定石。tool 定義は登録レイヤーで集約する。**Reference: `apps/mcp/code-graph-server/`**。

| severity | scope | 観点 |
|---|---|---|
| Critical | `apps/mcp` | 認証 / 認可が tool 単位で適切に適用されているか（書き込み系 tool が無認証で公開されていないか） |
| Major | `apps/mcp` | tool の実装が `tool-registry.ts` に集約され、`server.ts` から薄く呼ばれる構造になっているか |
| Major | `apps/mcp` | MCP ドキュメント（`docs/mcp/{name}.md`）が利用者向けに書かれているか（実装詳細を含めない、[document-writing.md](./document-writing.md) 参照） |

### Graph（`apps/graph/`）

データグラフ系アプリ（code、product、cortex-db、db-dictionary、screen、service-product）。比較的大規模で、`agent/` `ai/` `behavior/` `cli/` `graph/` のサブディレクトリに分割するのが定石。**Reference: `apps/graph/code/`**。

| severity | scope | 観点 |
|---|---|---|
| Major | `apps/graph` | グラフのビルド / 分析 / クエリの責務が別レイヤーに分離されているか |
| Major | `apps/graph` | BQ にロードする node / edge スキーマが型定義として明示されているか |
| Minor | `apps/graph` | グラフ更新の差分（incremental build）と全件再生成（full reindex）の両モードが選択可能か |

### Web（`apps/web/`）

フロントエンド・管理画面（cortex-admin、backoffice-console、mall 等）。React + Vite + TanStack Router 構成。`routes/` `components/` `hooks/` `lib/` `i18n/` の分割が定石。**Reference: `apps/web/cortex-admin/`**。

| severity | scope | 観点 |
|---|---|---|
| Critical | `apps/web` | 認証ゲート（OAuth / 認証ミドルウェア）が全 protected route に適用されているか |
| Critical | `apps/web` | API ベース URL / Secret 等の必須設定が起動時検証され、未設定でビルド or 起動が落ちる構造になっているか（[security.md](./security.md) の「必須設定とフォールバック禁止」参照） |
| Major | `apps/web` | ビジネスロジックがルート / コンポーネントに埋め込まれず、`packages/domain-*` または `lib/` の純粋関数に分離されているか |
| Major | `apps/web` | i18n キーが集約され、文字列ハードコードが含まれていないか |

## 共有パッケージ（`packages/`）

| 命名パターン | 役割 |
|-------------|------|
| `domain-*` | ビジネスドメインのロジック・定数・型 |
| `infra-*` | インフラ横断のヘルパー |
| `core-*` | 共通型定義・共有設定 |
| `*-core` | 特定技術の共通基盤 |

### 配置ルール

| 配置 | 用途 | 規約 |
|------|------|------|
| `packages/core/` | **真の全 app cross-cutting**（例: `@cortex/core-types`） | 新規追加は 30 以上の app/stack で利用される SoT 型・定数に限る。`packages/core/` 直下にサブパッケージを新規追加してはいけない（特定 app 群限定の contract / utility は `packages/domain/<bc>/` を使う） |
| `packages/domain/<bc>/` | bounded context ごとの domain logic + types | `*-contract` 命名は廃止。bounded context 名 + `src/types.ts` / `src/index.ts` に統一。caller が 1 つでも、特定 app 群に閉じた共有はここに置く |
| `packages/domain/<bc>/<sub>/` | bounded context 内のサブパッケージ（例: `domain/product/calendar-jp`） | bounded context が複数のユーティリティを束ねるときに使う |
| `packages/infra/<layer>/<name>/` | インフラ横断の共通ライブラリ | `@cortex/infra-*` 命名で stack 横断のフレームワーク的 helper を提供する |

`packages/core/` 直下の blanket regex（`.github/scripts/detect-changed-stacks.sh`）はすべての stack を trigger する設定なので、cross-cutting でないものをここに置くと過剰デプロイになる。配置先を迷ったら `packages/domain/<bc>/` を選び、新しい detect script case で trigger 対象 stack を明示すること。

| severity | scope | 観点 |
|---|---|---|
| Major | `packages/` | `@cortex/` 名前空間と `workspace:*` プロトコルを使用しているか（外部公開パッケージは下記例外を参照） |
| Major | `packages/core/` | 新規サブパッケージを追加していないか（30 以上の app で使われる SoT 型・定数でない限り `packages/domain/<bc>/` に置く） |
| Major | 全 app | ドメインロジックが app 内に埋め込まれず `packages/domain-*` に分離されているか |

### 名前空間の例外: 外部公開パッケージ

[docs/guidelines/package-publish.md](./package-publish.md) の publish-pattern を採用して GitHub Packages / npm registry に公開するパッケージは、`@cortex/` ではなく **`@air-closet/`** 名前空間を使用する。`@cortex/` は monorepo 内 workspace 専用の名前空間として残し、外部 consumer から install されうるパッケージは organization 名空間で区別する。具体例: `@air-closet/cc-analyzer`（`packages/cc-analyzer/`）。

## ロール / 権限設計

`@cortex/core-types` の `PortalRole` は **組織ポジションだけ** を表現する小さな enum である。
アプリ固有のアクセス権（特定 app を使えるかどうか）は中央の `PORTAL_PERMISSIONS` に増やさず、各 app 側で完結させる。

- [ ] `@cortex/core-types` の `PortalRole` に「アプリ名そのものを表すロール」を追加していないか（例: `*_resolver` / `*_manager`）
- [ ] アプリゲート（特定 app のアクセス可否）を `PORTAL_PERMISSIONS` に追加していないか。各 app の middleware と Firestore `app_access` コレクションで扱う
- [ ] 個別 email 単位の許可は `@cortex/infra-api-app-access` の `AppAccessRepository` + `isAppAccessGranted(...)` を経由しているか（独自に Firestore を読まない）
- [ ] Web 側で組織ロール再判定を行わず、API が返す `canAccess: boolean` をそのまま使っているか

`@cortex/infra-api-app-access` は、`line-user-id-resolver` で導入した app_access パターンを他の個別 app ゲートにも広げるための共通境界である。email 許可リスト読み取りのキャッシュ・同時ロード集約・ロールフォールバックを各 app で重複実装しないことを目的とする。

## データアーキテクチャ・Event-Driven パターン

| severity | scope | 観点 |
|---|---|---|
| Critical | BQ を使う app | パラメータ化クエリを使用しているか（SQL インジェクション対策） |
| Critical | Pub/Sub / 非同期処理 | メッセージの冪等性が確保されているか |
| Major | BQ を使う app | `SELECT *` を回避しているか、Embedding カラムは `mode: 'REPEATED'` を明示しているか |
| Major | `apps/api` | 長時間処理を同期 API 内で実行していないか（非同期に委譲すべき） |
| Minor | Firestore を使う app | コレクション命名の整合性、用途別ロールフィールドの区別 |

## インフラ / デプロイ

| severity | scope | 観点 |
|---|---|---|
| Critical | `infra/` | `Pulumi.prod.yaml` に `encryptionsalt` が含まれていないか |
| Major | `infra/` | Dockerfile に新しい `packages/` 依存の COPY + ビルドステップが追加されているか |
| Major | `infra/` | 変更検出スクリプトに新規スタックのパターンが追加されているか |
| Major | `infra/pages-*` | 新規 `pages-*` を追加した場合、`detect-changed-stacks.sh` の `add_pages` と `deploy-stack.yml` の `case "$PAGE" in` の両方が更新されているか（CI guard `verify-deploy-pages-integrity.sh` でも検証） |
