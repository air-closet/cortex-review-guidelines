# セキュリティ

## チェック観点

各項目に **severity** と **scope** を明記する。auto-reviewer は scope に該当しない PR では当該項目を発火させない。

### 認証 / 認可

| severity | scope | 観点 |
|---|---|---|
| Critical | `apps/api`, `apps/bot` | 認証エンドポイントに認証ミドルウェアが適用されているか |
| Critical | `apps/api`, `apps/bot` | 書き込み系（POST / PUT / DELETE）に適切なロールチェックがあるか |

### Secret / 機密情報

| severity | scope | 観点 |
|---|---|---|
| Critical | 全 app | API キー、パスワード、トークン、秘密鍵、Webhook secret 等の機密情報がソースコードにハードコードされていないか |
| Critical | 全 app | 機密情報の保存・配布に Secret Manager を使用しているか（パスワード・API キー・OAuth client secret・秘密鍵・Webhook secret 等） |
| Critical | Firestore を使う app | 機密情報を Firestore に**平文で保存**していないか。secret 本体ではなく Secret Manager の参照（例: `...SecretId`）だけを保存しているか |
| Critical | 全 app | secret / token / password / 個人情報をログ・エラーメッセージ・Slack 通知・Sheets 出力に出していないか |
| Critical | 全 app（docs / tests / samples 含む） | ドキュメント・テストデータ・サンプルコードに実物の secret を載せていないか（ダミー値のみ使用） |
| Major | `infra/` | Secret Manager を読む SA / IAM が最小権限になっているか（必要な runtime SA のみに `roles/secretmanager.secretAccessor` を付与） |

### IAM / 権限

| severity | scope | 観点 |
|---|---|---|
| Major | `infra/` | Cloud Run SA の権限が最小権限原則に従っているか |
| Major | `infra/` | SA に不要な IAM ロールが付与されていないか（未付与でサイレント失敗する実例あり — 過剰付与も問題） |

### 入力検証 / インジェクション対策

| severity | scope | 観点 |
|---|---|---|
| Major | `apps/api`, 外部入力を受ける app | ユーザー入力 / 外部 API 応答が型レベルのスキーマ（Zod 等）で**境界**で検証されているか |
| Critical | DB を使う app | SQL / NoSQL インジェクション対策: パラメータ化クエリを使用しているか |
| Major | Sheets API を使う app | `USER_ENTERED` 使用時に数式インジェクションリスクがないか（ユーザー入力を `=...` で始まる cell として書かない） |
| Minor | Sheets API を使う app | スプレッドシート ID がハードコードされていないか |

### 必須設定とフォールバック禁止

設定漏れを実行時の機能不全に変える「サイレントフォールバック」を境界で潰す。

| severity | scope | 観点 |
|---|---|---|
| Critical | 全 app | 必須設定（API host、Secret、環境変数等）に対して、未設定時に同一オリジン / 空文字 / `null` 等で**サイレントにフォールバックしていないか**。`baseUrl ?? ''`、`process.env.X ?? ''`、`?: string` の optional 化、`try/catch` でのデフォルト返却は禁止。必須なら型レベルで required にし、起動時に明示的に throw する |
| Critical | `apps/web` | `import.meta.env.VITE_*` を起動時に検証する関数（例: `getApiBaseUrl()`）が存在し、未設定で throw するか。コンポーネント / 関数の `baseUrl` / `endpoint` プロパティを optional にして同一オリジンへ落とす設計になっていないか |

検証は単に bundle / build を通すだけでなく、(a) 必須設定が欠けたらビルド / 起動時に確実に落ちる、(b) 通常起動時に設定値が末端まで配線される、の両方をテストで固定する。

### LLM / 外部 API クライアント

| severity | scope | 観点 |
|---|---|---|
| Major | LLM を使う app | LLM 呼び出しのエラーが握りつぶされていないか（サイレント失敗の原因になる） |
| Major | Vertex AI を使う app | Vertex AI SDK の初期化パラメータが正しいか（`vertexai: true` がないと Google AI Studio に向く） |

### Lint / 規約

| severity | scope | 観点 |
|---|---|---|
| Critical | 全リポジトリ | `eslint-disable` / `oxlint-disable` コメントが使用されていないか（cortex では絶対禁止） |
