# Cloud Run デプロイガイドライン

Cloud Run Job / Service のデプロイ設定に関する既知の罠と正しい対応。

## パターン1: `.gcloudignore` の `scripts/` 除外でルートアンカーを使う

### 問題

`scripts/` を unanchored（`scripts/`）で除外すると、BuildKit / gcloudignore の既知挙動で `apps/<x>/scripts/` まで巻き込まれて除外され、その後の `!apps/<x>/scripts/` 再インクルードが効かない。結果として `tsx apps/<x>/scripts/*.ts` をエントリポイントにする Cloud Run Job がコンテナ起動時に `ERR_MODULE_NOT_FOUND: Cannot find module ...scripts/*.ts` で死ぬ。

実例: biz-graph marketing-loader が `.gcloudignore` の unanchored `scripts/` により `apps/mcp/biz-graph-server/scripts/load-marketing-job.ts` を取り込めず、4/16 から 1ヶ月連続で起動失敗していた（`!apps/mcp/biz-graph-server/scripts/` を書いてあったが効いていなかった）。

### 対応

`.gcloudignore` も `.dockerignore` も **ルートアンカー `/scripts/`** で書く。これによりルート直下の `scripts/` のみが除外され、`apps/<x>/scripts/` は自動的に取り込まれるので `!apps/<x>/scripts/` の再インクルードは不要。

```text
/scripts/
```

### チェックリスト

Cloud Run Job を新規追加するとき:

- [ ] `.gcloudignore` / `.dockerignore` の `scripts/` 除外が `/scripts/` (ルートアンカー) になっている
- [ ] エントリポイント `tsx apps/<x>/scripts/*.ts` のスクリプトファイルが build context に含まれている（`.gcloudignore` のパターンをローカルで手動確認、または `gcloud builds submit` を実行してビルドログを確認 or デプロイ後の Job 1回テスト実行）
- [ ] Pulumi の Cloud Run Job 定義で参照するスクリプトパスと一致している

---

## パターン2: `pnpm deploy --legacy --prod` での推移的依存宣言漏れ

### 問題

PR #678 で導入した `pnpm deploy --legacy --prod` を使う Dockerfile パターンでは、**workspace パッケージの推移的依存であっても** `package.json` の `dependencies` に直接宣言しないと実行時に `ERR_MODULE_NOT_FOUND` になる。

`pnpm deploy` はデプロイ先に `node_modules` を生成するが、`--legacy` フラグを使う環境では推移的依存のホイスティングが限定的になり、直接 `import` しているにもかかわらず自 `package.json` に記載のないパッケージが見つからなくなる。

### 既知の違反 (潜在的)

- `apps/bot/libby-fassy` — `google-auth-library`
- `apps/graph/service-product/scripts/lib/metrics.ts` — `@google-cloud/firestore`

### 対応

`pnpm deploy --legacy --prod` を使う Dockerfile の app で `import` しているパッケージは、workspace 経由で解決可能であっても `package.json` の `dependencies` に直接追加する。

```jsonc
// ❌ 悪い例：workspace の共有パッケージが間接的に依存しているだけで、自分では宣言していない
{
  "dependencies": {
    "@cortex/otel": "workspace:*"
    // @google-cloud/bigquery は @cortex/otel の推移的依存だが未宣言
  }
}

// ✅ 良い例：直接 import するパッケージは自分で宣言する
{
  "dependencies": {
    "@cortex/otel": "workspace:*",
    "@google-cloud/bigquery": "catalog:"
  }
}
```

### チェックリスト

`pnpm deploy --legacy --prod` を使う Dockerfile の app を新規追加・修正するとき:

- [ ] ソースコードで直接 `import` しているパッケージが全て自分の `package.json` `dependencies` に宣言されている
- [ ] workspace パッケージを経由した推移的依存に依存していないか確認した

---

## パターン3: OTel exporter env の注入漏れ

### 問題

cortex の Cloud Run Service / Job は `apps/.../src/index.ts` の先頭で `initOtel({ serviceName: '...' })` を呼ぶ規約だが、`@cortex/otel` は `OTEL_EXPORTER_OTLP_ENDPOINT` と `GRAFANA_CLOUD_API_KEY` の双方が無いと初期化を skip する (`packages/infra/shared/otel/src/index.ts`)。

Pulumi 側でこの 2 つを `valueSource.secretKeyRef` で注入し忘れると、

- Grafana Cloud に trace / metrics / structured log が **何一つ送られない**
- 例外発生時に exception event が記録されないため、stdout に空 `error` だけが残り根本原因の特定が不可能になる
- `infra/observability/grafana-alert-rules.ts` の `Pipeline Sync Stale (...)` 系アラートは Grafana Cloud に届くログを前提にしているため、データソース不在で発火基準を満たせなくなる

`infra/pipeline/channel-talk` で実際にこのパターンを踏み、`channel-talk-analyzer` Cloud Run Job が 8 日間連続で失敗していたにもかかわらず Grafana から検知できない事案 (Tier 2 が 0 行のまま、`messages_masked` / `user_chats_safe` 空) が発生した。

### 対応

`infra/pipeline/<name>/index.ts` の Cloud Run Service / Job 定義に、core stack の Secret 参照を必ず注入する。

```ts
const grafanaOtlpEndpointSecretId = coreStack.getOutput(
  'grafanaOtlpEndpointSecretId',
) as pulumi.Output<string>;
const grafanaApiKeySecretId = coreStack.getOutput('grafanaApiKeySecretId') as pulumi.Output<string>;

// 該当 SA に secretAccessor を bind
new gcp.secretmanager.SecretIamMember(`${prefix}-<name>-grafana-endpoint-access`, {
  secretId: grafanaOtlpEndpointSecretId,
  role: 'roles/secretmanager.secretAccessor',
  member: serviceAccountMember(saEmail),
});
new gcp.secretmanager.SecretIamMember(`${prefix}-<name>-grafana-key-access`, {
  secretId: grafanaApiKeySecretId,
  role: 'roles/secretmanager.secretAccessor',
  member: serviceAccountMember(saEmail),
});

// envs に注入
{
  name: 'OTEL_EXPORTER_OTLP_ENDPOINT',
  valueSource: { secretKeyRef: { secret: grafanaOtlpEndpointSecretId, version: 'latest' } },
},
{
  name: 'GRAFANA_CLOUD_API_KEY',
  valueSource: { secretKeyRef: { secret: grafanaApiKeySecretId, version: 'latest' } },
},
```

参照実装: `infra/pipeline/jobcan/index.ts` の `secretEnvs` ブロック。

### チェックリスト

Cloud Run Service / Job を新規追加するとき:

- [ ] アプリ側 (`apps/.../src/index.ts`) が `initOtel({ serviceName: '...' })` を呼んでいる
- [ ] Pulumi 側 (`infra/.../index.ts`) で `OTEL_EXPORTER_OTLP_ENDPOINT` と `GRAFANA_CLOUD_API_KEY` を `secretKeyRef` で envs に注入している
- [ ] 該当 SA に対して両 secret の `roles/secretmanager.secretAccessor` を `gcp.secretmanager.SecretIamMember` で bind している
- [ ] デプロイ後にログで `[OTel] OTEL_EXPORTER_OTLP_ENDPOINT not set, skipping initialization` が出ていないことを確認した

---

## パターン4: `secretKeyRef version:latest` 参照時のバージョン不在による FAILED

### 問題

Cloud Run v2 は `secretKeyRef version:latest` を持つリビジョンを作成する際、対象 Secret に **バージョンが 1 件以上ない**とリビジョン作成が FAILED になる。

`createSecret` に `value` を渡さない外部 PAT やパスワード等の場合、Pulumi は Secret リソースのみ作成してバージョンを追加しない。そのままデプロイすると Cloud Run のリビジョンが FAILED になり、サービスが起動しない。

### 対応

`createSecret` に `value` を渡せない場合は、同じ `infra/*.ts` 内で `new gcp.secretmanager.SecretVersion(...)` に `secretData: 'PLACEHOLDER'` を `ignoreChanges: ['secretData']` 付きで作成し、初回デプロイを可能にする。デプロイ後は GCP Console から手動で実際の値に差し替える。

```ts
const mySecret = createSecret({
  name: 'my-external-pat',
  accessorServiceAccount: serviceAccount.email,
  // value を渡さない（ソースコードに書けない外部 PAT のため）
});

// Cloud Run の secretKeyRef は version が 1 件以上ないとリビジョン作成が失敗するため
// 初回デプロイ時にプレースホルダを作成する。
new gcp.secretmanager.SecretVersion(`${prefix}-my-external-pat-placeholder`, {
  secret: mySecret.id,
  secretData: 'PLACEHOLDER',
  deletionPolicy: 'DELETE',
}, { ignoreChanges: ['secretData'] });
```

デプロイ後、GCP Console の Secret Manager から対象 Secret を開き、新しいバージョンとして実際の値を手動で追加する。`ignoreChanges: ['secretData']` 設定により、以降の `pulumi up` ではこの値は上書きされない。

### チェックリスト

Cloud Run Service / Job で `createSecret` に `value` を渡さない Secret を `secretKeyRef` で参照するとき:

- [ ] 同じ `infra/*.ts` 内で `new gcp.secretmanager.SecretVersion(...)` に `secretData: 'PLACEHOLDER'` と `ignoreChanges: ['secretData']` を追加している
- [ ] デプロイ後に GCP Console から実際の値に手動で差し替えた
- [ ] 運用手順ドキュメントに「Pulumi が PLACEHOLDER を自動作成する → 手動で実値に差し替える → ignoreChanges で以降は上書きされない」の流れを記載した

---

## パターン5: `node:slim` 本番イメージでの `pnpm exec <cli>` 使用禁止

### 問題

`node:slim` ベースの本番 Docker イメージには `pnpm` が含まれない。アプリコードが実行時に `pnpm exec secretlint` 等の CLI ツールを `execFile` / `spawn` で起動しようとすると、`pnpm: not found` エラーで失敗する。

`bot-secretlint` 本番で `pnpm exec secretlint` を呼び出していたため、Cloud Run Service 上でスキャンが一切実行できない障害が発生した（PR #1011）。

### 対応

ランタイムで呼び出す CLI バイナリは、モジュールパス相対で `node_modules/.bin/<tool>` を直接解決する:

```ts
import { dirname, resolve } from 'node:path';
import { fileURLToPath } from 'node:url';

// ❌ 悪い例: pnpm がない node:slim イメージで失敗する
await execFile('pnpm', ['exec', 'secretlint', ...args], { cwd: workdir });

// ✅ 良い例: node_modules/.bin を直接参照する
const secretlintBin = resolve(
  dirname(fileURLToPath(import.meta.url)),
  '../node_modules/.bin/secretlint',
);
await execFile(secretlintBin, args, { cwd: workdir });
```

### チェックリスト

Cloud Run Service / Job でランタイムに CLI ツールを起動するとき:

- [ ] `execFile` / `spawn` / `spawnSync` の第一引数が `'pnpm'` になっていないか確認した
- [ ] バイナリを `node_modules/.bin/<tool>` への相対パスで解決している
- [ ] 解決先のパッケージが自 app の `package.json` の `dependencies` に直接宣言されている

---

## パターン6: `pulumi up --refresh` と `gcp-dev` ESC env の組み合わせ禁止

### 問題

`gcp-dev` ESC environment は `pulumiConfig.gcp:accessToken: ${gcp.login.accessToken}` を渡しており、Pulumi はこの値を default `pulumi:providers:gcp` リソースの `accessToken` input として **state に保存** する。

`pulumi up --refresh` の refresh phase は state 保存済み provider input から provider を再生成するため、前回 deploy 時に発行された access token (TTL 1h で expired) を使って GCP API を叩き、`googleapi: Error 401 ACCESS_TOKEN_EXPIRED` で即失敗する。

`pulumi up` (refresh なし) では provider input が deploy 前に update されるためこの問題は起きない。`pulumi refresh` 単体および `pulumi up --refresh` のみが破綻する。

token 自体の TTL は 1h あり expired ではない (新規発行 token を curl で直接叩くと普通に通る)。Pulumi がどの token を使うかという state semantics の問題であって、token / 認証側の問題ではない点に注意。

#1201 で全 deploy step に `--refresh` を付与したところ `gcp-dev` 参照 82 stack 全てが次回 deploy で失敗するようになり、#1206 / #1203 / #1202 で 3 stack の連続失敗が顕在化したため #1211 で revert した。

### 対応

- `.github/workflows/deploy-stack.yml` の `pulumi up` には `--refresh` を **付けない**
- drift 検知が必要な場合は別途 scheduled refresh job (週次 or 日次) として独立させ、deploy パスから切り離す
- `pulumi refresh` を必要とする場面 (例: `pending_operations > 0` 検知時) は `pulumi up --refresh` ではなく `pulumi refresh` 単体を **deploy 前に別ステップとして** 実行する。なおこのケースでも state 保存 token の問題は残るため、root cause を解消するには ESC env 側を変える必要がある

### 根本解消 (将来対応)

`gcp-dev` ESC env から `pulumiConfig.gcp:accessToken` を撤去し `environmentVariables.GOOGLE_OAUTH_ACCESS_TOKEN` のみで token を渡す形に変える (Pulumi 公式推奨)。env var は state に保存されないため refresh が壊れない。

ただし既存 82 stack の state に既に `accessToken` input が記録されているため、ESC 変更後に各 stack で `pulumi up` (refresh なし) を 1 回ずつ走らせて provider input を clean する移行作業が別途必要。

### チェックリスト

`.github/workflows/deploy-stack.yml` 等で `pulumi` を呼ぶとき:

- [ ] `pulumi up` に `--refresh` を付けていない
- [ ] drift 検知が必要なら deploy パスから独立した scheduled job として実装する
- [ ] 新しい ESC env を作るときは `pulumiConfig.gcp:accessToken` を使わず env var (`GOOGLE_OAUTH_ACCESS_TOKEN`) で渡す

---

## 参考

- [gcp-sdk-usage.md](./gcp-sdk-usage.md): Cloud Run + OTel での GCP SDK 利用制限
- [recurrence-prevention.md](./recurrence-prevention.md): 再発防止の必須アウトプット
