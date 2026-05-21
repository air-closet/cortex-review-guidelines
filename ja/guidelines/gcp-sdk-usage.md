# GCP SDK 利用ガイドライン

Cloud Run + Node 24 + OTel 環境では、GCP SDK の一部が `gaxios` / multipart upload / checksum 検証の既知問題を踏む。レビューでは直接 SDK 利用を Critical として扱い、metadata server で OAuth token を取得して REST API を `fetch` で直接呼び出す実装を正とする。

## 禁止

- `@google-cloud/storage` の新規 import。GCS upload で multipart boundary 混入、CRC32C / MD5 checksum mismatch、`gaxios` の `URL is required` を起こす。
- `@google-cloud/secret-manager` の新規 import。Cloud Run + OTel で認証経路が不安定になり、Secret Manager REST API 直接呼び出しのほうが実績のある正規パターンになっている。
- `@google-cloud/pubsub` の新規 import。Cloud Run + Node 24 + OTel 下では gRPC credential 取得が壊れて publish が `7 PERMISSION_DENIED: Method doesn't allow unregistered callers` で全件失敗する。channel-talk-pipeline で実際に踏み (5/14–5/17 の 4 run / 189 chunk が `publish_failed` で停止し Tier2 が 7ヶ月停止)、REST 化で復旧した。Cloud Run + OTel で Pub/Sub publish が必要な場合は `pubsub.googleapis.com/v1/projects/{p}/topics/{t}:publish` を `fetch` で直接叩く。

違反は Critical。既存の legacy import を見つけた場合も、新規・変更コードでは増やさず、触る機会に REST 実装へ寄せる。

## 正規パターン

1. `http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token` に `Metadata-Flavor: Google` を付けて OAuth token を取得する。
2. 対象サービスの REST API を `fetch` で直接呼び出す。
3. GCS upload は `POST https://storage.googleapis.com/upload/storage/v1/b/{bucket}/o?uploadType=media&name={encodedPath}` を使う。
4. GCS custom metadata は `x-goog-meta-{key}` header で渡す。
5. Secret Manager は `https://secretmanager.googleapis.com/v1/projects/{project}/secrets/...` を直接呼ぶ。

## 使用可

- `@google-cloud/bigquery` は現時点で cortex の Cloud Run / batch 実装に実績があり、禁止対象に含めない。

## 参考実装

- `apps/pipeline/jobcan/src/gcs.ts`: Jobcan CSV upload。checksum mismatch 回避の canonical 実装。
- `apps/pipeline/shopify/src/gcs.ts`: Cloud Run Node 24 の `gaxios` 問題を避ける GCS JSON API upload。
- `apps/pipeline/backlog/src/gcs.ts`: GCS JSON API upload と SDK checksum 問題の先例。
- `apps/pipeline/meet/backfill-gcs.ts`: 録画ファイル upload で multipart boundary 混入を回避。
- `apps/pipeline/db-account/src/secret-manager.ts`: Secret Manager REST API 直接呼び出し。
- `apps/pipeline/channel-talk/src/pubsub-rest.ts` (`createRestPublisher`): Pub/Sub REST publish の canonical 実装。metadata server token + `pubsub.googleapis.com/v1/projects/{p}/topics/{t}:publish` を `fetch` で直接叩く。

## 機械チェック

`packages/core-ecosystem-config/oxlint/index.ts` の共通 `no-restricted-imports` で `@google-cloud/storage` / `@google-cloud/secret-manager` / `@google-cloud/pubsub` の新規 import を禁止する。既存の legacy import は明示的な override に閉じ込め、追加する場合は override を増やさず REST 実装へ移行する。

## BigQuery: TIMESTAMP 列を再 insert する SQL では `CAST(... AS STRING)` を使わない

### 問題

BigQuery は TIMESTAMP 列を `CAST(ts AS STRING)` でキャストすると `2023-10-29 06:24:29.237+00` という **`+00` 短縮 timezone offset** 形式を返す。この文字列を別テーブル (TIMESTAMP 列) に streaming insert (`tabledata.insertAll`) で書き戻すと、サーバ側 parser が以下のエラーで reject する。

```text
Could not parse '2023-10-29 06:24:29.237+00' as a timestamp.
Required format is YYYY-MM-DD HH:MM[:SS[.SSSSSS]]
```

streaming insert の TIMESTAMP parser は `+HH:MM` 完全形 / `Z` / suffix なし / `UTC` 付き / epoch float は受け付けるが、**`+00` 短縮形だけ拒否する**（`sandbox_logs` で実測確認済）。

### 正規パターン

ETL の中間 SELECT で TIMESTAMP を文字列化して JS 側に渡し、後段で別 TIMESTAMP 列へ insert する場合は、必ず RFC3339 (`T...Z`) で整形する。

```sql
-- NG: streaming insert が "+00" を reject する
SELECT CAST(src.created_at AS STRING) AS created_at FROM ...

-- OK: RFC3339 を直接吐く
SELECT FORMAT_TIMESTAMP('%FT%H:%M:%E*SZ', src.created_at, 'UTC') AS created_at FROM ...
```

`%E*S` で trailing-zero を残さない秒精度（マイクロ秒精度のソースでも問題なく動く）、`Z` literal で UTC 明示。

### テスト fixture の落とし穴

`bigquery.query()` を mock するユニットテストでは、fixture を `'2026-04-10T00:00:00.000Z'` のような ISO 8601 で書きがちだが、これは production の `CAST(... AS STRING)` 出力 (`2023-10-29 06:24:29.237+00`) と異なる。**fixture が正常系と一致しているせいで、本番でしか発生しない insert reject を検知できない**。SELECT 文そのものを assert する regression test を入れる（例: `expect(query).toContain("FORMAT_TIMESTAMP('%FT%H:%M:%E*SZ'")`）。

### 参考実装

- `apps/pipeline/channel-talk-analyzer/src/etl.ts`: messages_raw / user_chats_raw → masked / safe テーブル ETL。`FORMAT_TIMESTAMP RFC3339` 使用。

## BigQuery: Policy Tag 保護カラムを JOIN キーにする MERGE は書込側 SA にも fine-grained reader が必要

### 問題

BigQuery の `MERGE INTO target USING source ON target.<col> = source.<col>` は、JOIN 述語に出てくる **target 側カラムを read として扱う**。当該カラムが Column-level Policy Tag (`pii_high` / `pii_medium` 等) で保護されている場合、書込専用に見える SA であっても `roles/datacatalog.categoryFineGrainedReader` を Policy Tag リソース単位で付与しないと以下で reject される。

```text
Access Denied: BigQuery: User has neither fine-grained reader nor masked get permission
to get data protected by policy tag "pii-classification-v1 : pii_medium"
on column <project>.<dataset>.<table>.<col>.
```

`WHEN MATCHED THEN UPDATE SET col = source.col` / `WHEN NOT MATCHED THEN INSERT (...)` 側は target カラムの read を伴わないため、JOIN 列以外の Policy Tag 列に対する fine-grained reader 付与は不要（最小権限を保てる）。

### 正規パターン

MERGE を実行する SA ごとに、JOIN キーとして登場する Policy Tag 保護カラムの **Policy Tag リソースに直接** `roles/datacatalog.categoryFineGrainedReader` を付与する（プロジェクトレベル付与は禁止）。

```typescript
// infra/core/channel-talk-iam.ts
new gcp.datacatalog.PolicyTagIamMember(`${prefix}-channel-talk-worker-pii-medium-reader`, {
  policyTag: piiMediumPolicyTag.name,
  role: 'roles/datacatalog.categoryFineGrainedReader',
  member: pulumi.interpolate`serviceAccount:${workerSa.email}`,
});
```

JOIN 列以外の Policy Tag 列（例: `pii_high` の `email` / `name`）には付与しない。Policy Tag を新規追加するとき / MERGE の JOIN キーを変更するときは、書込側 SA の付与をあわせて見直すこと。

### レビュー観点

新規 / 変更 SQL が `MERGE` を含む場合、レビュー時に以下を確認する:

1. `ON` 述語に出てくる target カラムが Policy Tag 保護対象か（`infra/core/data-catalog.ts` / dataset 定義で確認）。
2. 該当する場合、書込側 SA に `PolicyTagIamMember` で fine-grained reader が付与されているか（`infra/core/<domain>-iam.ts`）。
3. JOIN 列以外の Policy Tag 列には付与しないこと（最小権限）。

### 参考実装

- `infra/core/channel-talk-iam.ts`: analyzer SA に `pii_high` / `pii_medium`、worker SA に `pii_medium` のみを付与。worker は `mergeUsers` の JOIN target カラム (`users_raw.user_id`) read 経路のためだけに `pii_medium` を持つ。

## BigQuery: 名前付きクエリパラメータで TIMESTAMP / DATE / DATETIME を渡すときは `bqTimestampParam()` を使う

### 問題

`@google-cloud/bigquery` の `query()` / `createQueryJob()` に名前付きパラメータを渡すとき、生の ISO 8601 文字列を `params` に入れて `types` で temporal type を明示すると、クライアントの serializer が wire format の `parameterValue.value` を **`undefined`** にしてしまう。BigQuery はこれを **NULL** として解釈し、`INSERT INTO ... <REQUIRED column>` で以下のように reject する。

```text
Required field <column> cannot be null
```

```typescript
// NG: parameterValue.value が undefined になり BQ が NULL 扱いする
await client.query({
  query: `INSERT INTO t (ts) SELECT TIMESTAMP(@ts)`,
  params: { ts: '2026-05-13T18:43:36.544Z' },
  types: { ts: 'TIMESTAMP' },
});
```

`BigQuery.valueToQueryParameter_('2026-05-13T18:43:36.544Z', 'TIMESTAMP')` の戻り値は
`{ parameterType: { type: 'TIMESTAMP' }, parameterValue: { value: undefined } }`。

### 正規パターン

temporal パラメータは `@cortex/bigquery` の `bqTimestampParam()` で `BigQueryTimestamp` インスタンスに変換して渡す。`BigQueryTimestamp` は型を自己記述するため `types` の明示指定自体が不要になり、SQL 側の `TIMESTAMP(@ts)` cast も不要。

```typescript
import { bqTimestampParam } from '@cortex/bigquery';

// OK: parameterValue.value に ISO 文字列が入り、parameterType は TIMESTAMP に自己記述される
await client.query({
  query: `INSERT INTO t (ts) SELECT @ts`,
  params: { ts: bqTimestampParam('2026-05-13T18:43:36.544Z') },
});
```

### 自動検証

- ESLint ルール `graph/no-bq-string-timestamp-param` が `query` キーを持つ object literal の `types` に `TIMESTAMP` / `DATE` / `DATETIME` / `TIME` の明示指定を検出する（既存違反の移行中は `warn`、解消後 `error` へ昇格）。
- mock-only のユニットテストでは wire format の欠落を検知できない。`@cortex/bigquery` の contract test (`BigQuery.valueToQueryParameter_` の戻り値を assert) で serializer レベルの regression を固定する。

### 参考実装

- `packages/infra/shared/bigquery/src/index.ts`: `bqTimestampParam()` 本体と wire format contract test。
- `apps/pipeline/channel-talk/src/analyzer-enqueue.ts`: `insertAnalyzerRun` / `insertChunkItems` で `bqTimestampParam()` を使用。

## BigQuery: SELECT エイリアスと同名カラムの ORDER BY で `SUM()` を二重包みしない

### 問題

`SELECT SUM(col) AS col ... GROUP BY ... ORDER BY SUM(col)` のように、SELECT でエイリアス名と元カラム名を同名にした場合、ORDER BY 内で再び `SUM(col)` と書くと、BigQuery が「エイリアスの集計」（= `SUM(SUM(col))`）として解釈し、ネストした集計エラーになる場合がある。

```sql
-- NG: ORDER BY の SUM(impressions) が SELECT alias をラップして nested aggregate になりうる
SELECT
  appeal_axis,
  SUM(impressions) AS impressions
FROM t
GROUP BY appeal_axis
ORDER BY SUM(impressions) DESC

-- NG: 複合式でも同様
SELECT
  SUM(input_tokens) AS input_tokens,
  SUM(output_tokens) AS output_tokens
FROM t
GROUP BY nickname_key
ORDER BY (SUM(input_tokens) + SUM(output_tokens)) DESC
```

### 正規パターン

ORDER BY ではエイリアスを直接参照する（`SUM()` で再度包まない）。

```sql
-- OK: 単一カラムはエイリアス直接参照
ORDER BY impressions DESC

-- OK: 複合式もエイリアスの算術演算で参照
ORDER BY (input_tokens + output_tokens + cache_creation_input_tokens + cache_read_input_tokens) DESC
```

### 参考実装

- `apps/api/product/src/features/cc-usage/infrastructure/bigquery-cc-usage-repository.ts`: `getUserBreakdown` / `getRepositoryBreakdown` の修正例。

## BigQuery: `createQueryJob` は `getQueryResults()` / `promise()` で完了を待つ

### 問題

`@google-cloud/bigquery` の `BigQuery#createQueryJob()` は **job リソースの作成までしか完了させない**。返却された `Job` インスタンスは BQ 側で実際にクエリ / DML が走り切る前の状態である。

```ts
// NG: job 作成完了は待つが、DML 実行完了は待たない
const [job] = await client.createQueryJob({ query: 'INSERT INTO t SELECT ...' });
const [metadata] = await job.getMetadata(); // numDmlAffectedRows は undefined の可能性
return Number(metadata.statistics?.query?.numDmlAffectedRows ?? 0); // 0 として扱われる
```

- 直後に **同じテーブルを read** すると read-after-write が見えず空 snapshot を返す。
- `getMetadata()` は単なる snapshot 取得で、ジョブ完了同期はしない。
- BQ 側で実行完了するまでの数秒〜数十秒の間、後続処理は破壊的に進む。

この罠を踏んで chunks_planned=0 で Pub/Sub publish 0 件 (analyzer 起動せず) になった事故が `apps/pipeline/channel-talk/src/analyzer-enqueue.ts` で発生した。

比較対象として `BigQuery#query()` は内部で `createQueryJob` → 完了待ち → rows 返却を行う convenience method。他クエリでは完了同期に見えていたのはこのため。

### 正規パターン

`createQueryJob` で取得した `Job` には**必ず `getQueryResults()` または `promise()` を `await` する**。`numDmlAffectedRows` 等の metadata だけ欲しい場合でも、完了待ち後に `getMetadata()` を呼ぶ。

```ts
const [job] = await client.createQueryJob({ query: 'INSERT INTO t SELECT ...' });
await job.getQueryResults(); // ← DML 完了を待つ
const [metadata] = await job.getMetadata();
return Number(metadata.statistics?.query?.numDmlAffectedRows ?? 0);
```

`getQueryResults()` は rows を返すが、DML / DDL の場合は空配列が返る。完了同期の副作用が目的なので戻り値は捨てて良い。

`job.promise()` は内部で `getQueryResults()` を呼ぶ薄いラッパー。意味的にどちらでも同等。

### 自動検証

- ESLint ルール `graph/no-bq-create-query-job-without-await-completion` が `createQueryJob` 呼出しの戻り `Job` binding に対し `getQueryResults` / `promise` の member access が scope 内に存在することを ESLint scope manager で検証する。binding を `return` する wrapper パターンは許可。
- mock-only のユニットテストでは real BQ の async 挙動が再現されず検知できない。invocation 順序 (`getQueryResults` 解決前に次の SELECT を発行していないこと) を `vi.fn` の呼び出し順で assert する regression test を入れる。例: `apps/pipeline/channel-talk/src/analyzer-enqueue.test.ts` の「insertChunkItems の DML 完了 (getQueryResults) を待ってから insertChunks の SELECT を発行する」。

### 参考実装

- `apps/pipeline/channel-talk/src/analyzer-enqueue.ts` (`insertChunkItems`): `createQueryJob` → `getQueryResults` → `getMetadata` の正規パターン。
- `apps/pipeline/channel-talk-analyzer/src/merge.ts`: MERGE 文の完了待ち。
- `packages/eslint-plugin-graph/src/rules/no-bq-create-query-job-without-await-completion.ts`: lint ルール本体。
