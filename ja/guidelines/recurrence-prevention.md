# 再発防止ガイドライン

障害 / 不具合の修正で「同じバグを 2 度起こさない」ための判断基準。bug fix を**点**で終わらせず**面**に拡張する。

## 思想

bug fix の目的は「対象の症状を消すこと」ではなく「再発のクラスを閉じること」。同じ罠に 2 度はまったら、それはコード品質ではなくチームの仕組みの問題として扱う。

[document-writing.md](./document-writing.md) の優先順位（型 > lint > test > 手書き）と同じく、**機械化できるものは機械化する。テキストで規律を保とうとしない。**

「とりあえず注意点メモに書く」「TODO コメントを残す」「あとで対応」は禁止する。これらは [document-writing.md](./document-writing.md) の「ドキュメントの価値は鮮度に依存する」原則に反し、必ず陳腐化する。

## 「将来的に対応」を禁止する

再発防止アクションを **その時点で** 完了させる。次のような将来送りの表現は禁止する。

- 「将来的に lint 化を検討」「将来的に強化」「将来的に error 昇格」
- 「次回のリファクタリング時に検討」「次フェーズで対応」「別 PR で対応」
- 「既存違反が残るため `warn` で導入し、後段で `error` へ昇格」
- 「TODO: ... する」「FIXME: ...」のコード内残置

対応可能なら **その PR で対応する**。lint ルールを追加するなら同 PR で既存違反も 0 にして `error` で投入する。既存違反が多すぎて根本対応が現実的でない場合は、lint ルール導入自体を当該 PR では行わず、根本対応 PR と同時に投入する（分割するなら根本対応 → ルール導入の順、ルールは初手から `error`）。

「既存違反が残るため `warn` で導入」というロールアウト手法は **採用しない**。これは事実上の deferral であり、`warn` を `error` に昇格する責務が宙に浮いて陳腐化する。`error` で投入できない時点で、PR スコープを根本対応まで含めて再設計する。

## 判定マトリクス

bug fix PR では下記を順に判定し、該当するアクションを **同 PR 内** で実行する。

| 状況 | 必須アクション | 形式 |
|------|---------------|------|
| 同じ罠を 2 回以上踏んだ | **lint 化を必須**（カスタム ESLint ルール / 型制約 / CI ガード） | 機械化 |
| パターンが他にも存在しうる | **横展開を必須**（Product Graph で類似ノード走査、既発見箇所も同 PR で修正） | 調査 + 修正 |
| 機械検証不能だが原則として価値あり | **既存 guideline に項目追加**（新ファイル作成は最後の手段） | guideline |
| 単発・原則化価値なし | **何もしない**（bug fix のみ。必要なら git log / blame で再発見） | — |

## 各アクションの判断基準

### lint 化を必須とする条件

下記いずれかに該当すれば lint 化必須:

- **同一パターンの罠が 2 回以上発生している**（git log / blame で確認）
- **新規ファイル追加時に高頻度で発生する罠**（命名規則、import 制約、インフラ宣言の漏れ等）
- **AST レベルで検出可能な禁止パターン**

実装手順は [lint-rules.md](./lint-rules.md) に従う。`no-restricted-syntax` / `no-restricted-imports` の直書きは禁止で、`@cortex/eslint-plugin-graph` のカスタムルールとして実装する。

### 横展開を必須とする条件

下記いずれかに該当すれば、同 PR 内で他箇所を走査・修正:

- **共通パターン（命名・import・初期化）に従う関数や設定が複数存在する**
- **同じデータソース・API・テーブルを参照する複数のスタックがある**
- **同じ Pub/Sub トピック・Cloud Scheduler ジョブを共有する**

走査は Product Graph MCP を起点とする:

```text
search_product_graph_nodes(query: "<罠の特徴>", search_mode: "semantic")
trace_product_graph_connections(start_node: "<該当ノード>", direction: "both")
```

詳細は [impact-analysis.md](./impact-analysis.md) を参照。

### guideline に項目追加する条件

機械検証は不可能だが、原則として将来の判断に影響するもの:

- 設計判断（DDD 層配置、共有 package 切り出し基準等） → [architecture.md](./architecture.md)
- セキュリティ原則（認証境界、Secret 取り扱い等） → [security.md](./security.md)
- テスト品質（matcher 選択、flaky 排除等） → [testing.md](./testing.md)
- ドキュメント運用 → [document-writing.md](./document-writing.md)

**新規 guideline ファイルは最後の手段**。既存の guideline に組み込めないか先に検討する。新ファイルが乱立すると相互参照が破綻する。

### 「何もしない」を選ぶ条件

下記**全て**に該当すれば、bug fix のみで完結させる:

- 単発の発生で、再発可能性が低い
- 機械化が技術的に困難（外部 API のレートリミット変動、third-party 依存等）
- 既存 guideline に組み込めない（汎化が無理 / scope が狭すぎる）

ドキュメント化しないことを明確に選ぶ。「念のため記録に残しておく」は陳腐化の起点。

## 適用例

### 例 1: `@google-cloud/storage` / `@google-cloud/secret-manager` の新規 import 禁止

**罠**: Cloud Run + Node 24 + OTel では GCP SDK が trace / metrics 送信を阻害する。GCS / Secret Manager は metadata server で OAuth token を取得して REST API を `fetch` で直接叩く必要がある。

**判定**: 過去複数の app で同じ罠を踏んでおり、import 1 行で誤りが発生する → **lint 化必須**

**アクション**: `packages/core-ecosystem-config/oxlint/index.ts` の共通 `no-restricted-imports` で禁止済み。詳細は [gcp-sdk-usage.md](./gcp-sdk-usage.md) を参照。新規禁止対象を追加する場合は [lint-rules.md](./lint-rules.md) に従う。

### 例 2: `.gcloudignore` 再インクルードの漏れ

**罠**: `tsx scripts/*.ts` を Cloud Run Job のエントリポイントにする app を追加するとき、`.gcloudignore` に `!apps/<path>/scripts/` 再インクルードがないとデプロイ後に script が見つからず即時死亡する。同一パターンで 4 件発生。

**判定**: 4 件の再発がある + 機械検証可能（`.gcloudignore` パース + Cloud Run Job Pulumi 定義の cross check） → **lint 化または CI script 化必須**

**アクション**: `scripts/check-gcloudignore-consistency.ts` を CI ガードとして実装済み（`.github/workflows/test.yml` の guards job で実行）。`apps/<path>/scripts/` の再インクルードが必要な app を src / Dockerfile から自動検出し、`.gcloudignore` に欠けている場合は CI が落ちる。詳細は [cloud-run-deploy.md](./cloud-run-deploy.md) パターン1 を参照。

### 例 3: OTel exporter env 注入の漏れ

**罠**: 新規 Cloud Run Service / Job の Pulumi 定義で `OTEL_EXPORTER_OTLP_ENDPOINT` と `GRAFANA_CLOUD_API_KEY` を `secretKeyRef` で envs に注入し忘れると、`@cortex/otel` が init を skip し trace / log / metrics が Grafana に送られず**障害が検知不能**になる。

**判定**: 機械検証可能（Pulumi resource 解析）→ **lint 化済み**

**アクション**: `scripts/check-otel-env-injection.ts` を CI ガードとして実装済み（`.github/workflows/test.yml` の guards job で実行）。`infra/**/*.ts` で `gcp.cloudrunv2.Service` / `gcp.cloudrunv2.Job` を構築し、参照 app の top-level entrypoint が `initOtel()` を呼んでいる場合に `OTEL_EXPORTER_OTLP_ENDPOINT` / `GRAFANA_CLOUD_API_KEY` の宣言を要求する。詳細は [cloud-run-deploy.md](./cloud-run-deploy.md) パターン3 を参照。

既存 violations は `ALLOWLIST` に明示記録され、個別 stack の Pulumi PR で順次解消する（新規違反は CI で阻止）。

### 例 4: BigQuery 名前付き TIMESTAMP パラメータの bind ミス

**罠**: `query()` / `createQueryJob()` に `params: { ts: isoString }` + `types: { ts: 'TIMESTAMP' }` の形で生 string を渡すと、`@google-cloud/bigquery` の serializer が wire format の `parameterValue.value` を `undefined` にし、BigQuery が NULL として解釈する。`INSERT INTO ... <REQUIRED column>` が `Required field ... cannot be null` で reject され、channel-talk の Tier2 ETL が本番で停止した。`bigquery.query()` を完全 mock するユニットテストでは SQL も parameter binding も実行されないため検知不能だった（同種の「mock fixture の落とし穴」は [gcp-sdk-usage.md](./gcp-sdk-usage.md) で既出）。

**判定**: AST レベルで検出可能な禁止パターン（`types` に temporal type 文字列）+ 既存 guideline に組み込める → **lint 化 + guideline 追記**

**アクション**: ESLint ルール `graph/no-bq-string-timestamp-param` を導入（`types` に `TIMESTAMP` / `DATE` / `DATETIME` / `TIME` の明示指定を検出、`error` レベル）。共通ヘルパー `@cortex/bigquery` の `bqTimestampParam()` で `BigQueryTimestamp` インスタンスとして渡す正規パターンを [gcp-sdk-usage.md](./gcp-sdk-usage.md) に追記。serializer レベルの regression は `@cortex/bigquery` の contract test で固定。channel-talk / github-activity-core / member-daily-report-core / basic-design-archive / db-account / meet を含む既存違反は同 PR で全件 `bqTimestampParam()` へ移行済み。

### 例 5: try/catch および Promise#catch のサイレント握りつぶし

**罠**: `catch (e) {}` / `.catch(() => null)` / `.catch(() => [])` のような握りつぶしパターンは AI 生成コードに頻出し、エラー発生時に観測できないまま処理が継続するため、本番障害が「件数も原因も分からない」状態で進行する。NotFoundError 等を型名だけで warn に下げる「型名ベースのレベル降格」も同じ問題を生む。

**判定**: AST レベルで検出可能（`catch` ボディの中身解析 / `.catch` 呼び出しの handler 解析）+ ガイドラインで体系化可能 → **lint 化 + guideline 追加**

**アクション**: ESLint ルール `graph/no-silent-catch` を導入（`error` レベル）。空 catch ブロックおよび Promise#catch でのリテラルフォールバック / 空ボディハンドラを検出し、`@cortex/otel/logger` での構造化ログ出力または再 throw を強制する。エラーログの判定基準（fatal / error / warn / info / debug）は [observability.md](./observability.md) の「ログレベルの判定基準」に集約。導入時に既存 39 件の違反は全件同 PR で根本修正済み（再 throw / `serializeError` 構造化ログ / ENOENT / SyntaxError discriminator 等）。

### 例 6: db-account-pipeline 新規ターゲット追加時のセットアップ漏れ

**罠**: db-account-pipeline に新しい DB ターゲットを追加するとき、以下 2 種類の **値投入漏れ** が手動チェックリストに依存しており、コード上は静的に検出できないため発生しやすい。

1. **踏み台 SSH 秘密鍵 Secret の version 未投入**: `infra/pipeline/db-account/index.ts` の `BASTION_KEY_SECRETS` には Secret 名が宣言されるが、`createSecret()` ヘルパーは値（PEM）を引数に取らない設計のため、Pulumi で作られる Secret は空シェルのまま放置される。手動 `gcloud secrets versions add` を忘れると、Pipeline 自体は起動・処理を続けるが、**SSH 公開鍵を伴う通常申請（非 AI-only）が来た瞬間に** `[DB] Lambda executor` 直前の `getSecret(...adminKeySecret)` が `Secret ... not found or has no versions` で 404 を返し、当該 DB が部分失敗する。AI-only 申請しか来ない期間は完全に潜伏し、`db_account_processed` テーブルも `completed` のままなので異常検知できない（実例: ecosale-prod が 2026-03-31 のターゲット追加から 2026-05-18 の初の通常申請まで 7 週間以上気付かれなかった）。
2. **管理スプレッドシートのタブ未作成**: `db-configs.ts` の `(spreadsheetId, sheetNamePrefix)` に対応する `{prefix}_view` / `{prefix}_edit` / `{prefix}_delete` / `{prefix}_stg` のタブが既存スプレッドシートに無いと、処理は `[Processor] Spreadsheet append failed: Unable to parse range` の非クリティカル WARN を吐いて `completed` を返す。申請者には Firestore / Google Docs / Slack DM で credential が届くため**気付けない**が、管理シート上で誰がどの権限を持つか追跡できなくなる。

**判定**: AST レベルでは検出できない。Pulumi state / Sheets API への query で外形検証は技術的に可能だが、定常 CI への組込には GCP state バックエンド認証・Sheets API クレデンシャルの CI 注入が必要で基盤依存が過大なため **CI ガード化は意図的に非採用**。1 で 1 回（ecosale-prod）、2 で 4 件（ecosale_{view,edit,delete} + phoenix_styling prod tabs）の同時露呈 = **横展開対象**。新規 DB 追加時の手動チェックリストに依存している項目を**対話式スクリプトに組込 + guideline 明文化**することで対応する。

**アクション**:

- `scripts/add-db-account-target.ts` を拡張: 新規 VPC グループの踏み台 SSH 鍵を対話式で投入する（PEM ファイルパスを受け取って `gcloud secrets versions add` + Pipeline SA への IAM binding まで実行する）。値投入をスキップした場合は稼働前に必ず投入するよう `gcloud secrets versions add` コマンドを警告ログで案内する。
- スクリプト末尾の手動チェックリストに **「管理スプレッドシートのタブ作成」** を独立項目として追加。prod / stg ごとの必要タブ名、既存タブからのヘッダー行コピー手順、未作成時の潜伏挙動を明示する。
- `docs/pipeline/db-account.md` の運用手順「新規データベースの追加」を更新し、上記 2 点を必須項目化、AI-only のみで潜伏する性質を `IMPORTANT` callout として記録する。

### 例 7: 削除した workspace package への workflow `--filter` 参照残り

**罠**: monorepo の workspace package を削除する PR で、`.github/workflows/*.yml` の `pnpm exec turbo run build --filter='@cortex/<pkg>'` 等の filter からの参照を消し忘れると、その filter が走る stack の deploy / build が `x No package found with name '@cortex/<pkg>' in workspace` で失敗する。普段 build されない stack（例: aws stack の Lambda build）では休眠し、次に該当 stack が trigger されるまで気付かない。`feat(line-user-id-resolver): Phase 5 — drop dedicated Lambda` (#974) で `@cortex/line-user-id-resolver-executor` を削除した際に `deploy-stack.yml` の filter から消し忘れ、aws stack deploy で初めて表面化した (PR #1006)。

**判定**: 機械検証可能（workflow YAML 内 `--filter` トークン解析 + `pnpm list -r --json` の cross check）→ **CI script 化必須**

**アクション**: `scripts/check-workflow-filter-references.ts` を CI ガードとして実装済み（`.github/workflows/test.yml` の guards job で実行）。`.github/workflows/*.{yml,yaml}` 内の `--filter[ =]['"]?@cortex/<name>['"]?`（turbo の `...` / `^` / `[ref]` 修飾も対応）を抽出し、`pnpm list -r --depth=-1 --json` で workspace に実在しないものを検出する。新規違反は PR 時点で CI が落ちる。

## レビュー時のチェック観点

bug fix PR をレビューするときは:

- [ ] PR description に再発防止アクション（lint / 横展開 / guideline 追加 / 何もしない）が明記されているか
- [ ] 「何もしない」選択時、上記の全条件を満たしているか reviewer が確認したか
- [ ] 横展開アクションを取った場合、Product Graph での走査範囲と発見箇所が PR description に記載されているか
- [ ] 追加された lint ルール / guideline 項目が既存のものと重複していないか

これを満たさない障害修正は再発防止不足として Major 以上で指摘する。データ破壊、認証不備、または本番ジョブ停止につながる再発可能性がある場合は Critical とする。
