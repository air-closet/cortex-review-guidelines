# 観測性・通知メッセージ作成ガイドライン

cortex のログ・Slack 通知・アラート本文・運用通知は **省略・truncate を禁止**し、
**エラー発生時は適切なレベルで構造化ログを必ず出力する**。
観測性は事故・劣化・閾値割れの検知と一次切り分けの土台であり、本文を切ったり、
エラーをサイレントに握りつぶしたりすると「何件起きたか」さえ分からなくなり、
原因特定を別経路で再調査することになる。通知・ログを受け取った人がそのメッセー
ジだけで一次対応を完結できる粒度を維持する。

このガイドラインは大きく 2 つの観点を扱う:

1. **通知・ログ本文の truncate 禁止**（全件出す / 分割送信する）
2. **エラーハンドリングとログレベル**（サイレント catch 禁止 / 適切なレベルで出力する）

---

## 1. 通知・ログ本文の truncate 禁止

### 禁止する処理

以下のような形で通知・ログ本文の情報量を削るのは禁止する。

- `slice(0, N)` / `take(N)` / `head(N)` で配列を打ち切る
- 「…他 N 件」のような省略 suffix を末尾に足して切り捨てる
- `JSON.stringify` の depth 制限・`length` 上限で本文を縮める
- `String#substring` / `String#slice` で本文末尾を欠落させる
- `console.log` / logger 出力をループ外でまとめて 1 行にして件数だけ出す

### 守るべき原則

1. **全件出す**: breach / 失敗 / 検出されたエラーは件数に関係なく全件列挙する。
   人間が読むのは大変でも、機械が grep / コピーするときに必要。
2. **必要なら分割送信**: Slack / Discord 等のメッセージ長に上限がある通知先は、
   省略ではなく **複数メッセージに分割して全量送る** こと（`chat.postMessage` を
   ループする / thread にぶら下げる等）。
3. **構造化ログを優先**: 大量データはテキストではなく構造化フィールド
   (`logger.info({ items: [...] }, 'message')`) で出す。観測ツール側で必要に応じ
   集約 / フィルタする。
4. **件数サマリは併記、置き換えではない**: `count`, `min`, `max` のサマリは
   全件の前後に併記してよい。サマリだけで全件を置き換えてはならない。

### 例外

- **PII / 機密情報のマスキング**: 個人情報・credentials・PII を含むフィールド
  はマスクする。これは「情報の削減」ではなく「機密の防護」なので本ガイドラインの
  対象外。マスク後の値（`***` 等）と元の構造は維持すること。
- **物理上限の回避**: Slack の text 4 万文字、Cloud Logging エントリの payload
  上限 (256KB) など、技術的にどうしても入らない場合は分割送信する。Truncate
  する場合は `<<truncated: see ...>>` のように切れた事実とフル参照先（GCS / BQ /
  Logs Explorer URL 等）を併記し、人間が一次対応を完結できる導線を残す。

---

## 2. エラーハンドリングとログレベル

エラーが発生したときに **下手にフォールバックしてサイレントに握りつぶさず、
意味に応じた適切なログレベルで構造化ログを出す**。
AI 生成コードは「とりあえず動かす」ために catch して空値を返す / null を返す /
握りつぶす傾向があり、結果として本番障害が観測されないまま劣化する。

### 禁止する処理

| severity | 観点 |
|---|---|
| **Critical** | `catch (err) {}` / `catch { return null; }` / `catch { return []; }` / `.catch(() => undefined)` のように、エラー情報を出力せずに正常値を返して握りつぶす |
| **Critical** | `if (!result) return;` で本来エラーであるべき欠損をサイレントに skip する |
| **Critical** | 必須リソース（DB レコード、設定、依存サービスの応答）が無い場合に warn 以下で済ませて処理を継続する |
| **Major** | `logger.error(err.message)` のように `Error.stack` / `cause` を捨てて文字列だけ残す |
| **Major** | `logger.error('Failed')` のように context（処理対象 ID・パラメータ・ステージ）を付けずに固定文字列だけ出す |
| **Major** | catch して `console.error` / `console.log` で出力する（Pino logger でなく素の console を使う） |
| **Minor** | catch ブロックで `// ignore` / `// best-effort` 等のコメントだけ書いて、本当に無視してよい理由を PR description / コメントで説明していない |

### ログレベルの判定基準

エラーレベルは「Error 型かどうか」「予期されているかどうか」では決めない。
**そのエラーがその機能にとってどういう意味を持つか** で判定する。
`NotFoundError` のような型名や `expected` というラベルを根拠に warn に下げない。
例えば「絶対に存在すべきレコードが存在しない」状況は warn ではなく error / fatal。

| レベル | 判定基準 | 例 |
|---|---|---|
| **fatal** | その機能全体が **20% 以上失敗する状態** になることが予想される。サービス継続不能・致命的な設定欠如・依存先の全断 | OTel 初期化失敗 / 起動時の必須 secret 欠如 / Pipeline の入力データソース全断 |
| **error** | 確実に **後からデータ復旧 / 再実行が必要**。影響範囲は 20% 未満と予想される。本来存在すべきデータの欠損、本来成功すべき外部 API 呼び出しの失敗 | 「あるはずの user レコードが見つからない」/ BigQuery insert 失敗 / 個別レコードの enrichment 失敗 |
| **warn** | **業務上予見されうるもので、ただちに問題にならない**。リトライで自然回復する想定の transient エラー、ユーザー起因の入力 validation 失敗、想定済みの空結果 | 検索クエリで該当 0 件 / 任意フィールド未設定 / レート制限による短時間 retry |
| **info** | 通常運用のイベント（開始 / 完了 / 件数） | バッチ完了サマリ / リクエスト処理ログ |
| **debug** | 開発時の詳細トレース。本番で常時出力しない | クエリ本文 / 中間状態 |

判定フロー:

1. このエラーで **データを後から復旧する必要があるか?** YES → error 以上
2. このエラーが **広範に発生し、機能全体の SLO を割る可能性があるか?** YES → fatal
3. 業務的に **想定済みで、リトライ / 次回で自然回復する** か? YES → warn
4. それ以外で、復旧不要・継続可能なイベント → info / debug

「`NotFoundError` だから warn」「`ValidationError` だから warn」のような型名ベース
の機械判定は禁止。同じ `NotFoundError` でも「絶対存在すべきものの欠損」なら
error / fatal、「ユーザー検索のヒット 0 件」なら warn になる。

### 構造化ログの最低要件

catch してログを出す場合、以下を満たす構造化ログ (`@cortex/otel/logger` の
Pino logger) を使う。

```typescript
import { createLogger, serializeError } from '@cortex/otel/logger';

const logger = createLogger('service-name');

try {
  await doSomething(targetId);
} catch (error) {
  logger.error(
    {
      err: serializeError(error),
      event: 'do_something_failed',
      targetId,
      stage: 'enrichment',
    },
    'Failed to enrich target',
  );
  throw error; // または業務上の代替値で継続する場合は理由を明記する
}
```

必須要件:

- **`err` フィールドに `serializeError(error)` の結果**: `name` / `message` / `stack` が構造化フィールドとして残る。`err.message` だけ載せて stack を捨てない
- **context フィールド**: 処理対象の ID・ステージ・関連パラメータを構造化フィールドで添える。`'Failed: ' + JSON.stringify(...)` のような文字列結合は不可（grep / Loki クエリで集計できない）
- **`event` フィールド**: 後段の Loki / Grafana クエリで集計できるよう、固定の event 名を付ける（snake_case 推奨）
- **メッセージは固定文字列**: メッセージ本体に動的値を埋め込まない（`Failed to enrich user ${userId}` ではなく `'Failed to enrich user'` + `{ userId }`）

### 再 throw / 上位ログ委譲

catch ブロックで **同じエラーを再 throw する場合はログ不要**（上位 handler / 起動
時の `process.on('uncaughtException')` が出す前提）。次のいずれかを満たす:

1. **同じ error を再 throw する**: catch せず上に伝播させるのと同義なので、追加ログ不要
2. **error を変換して throw する**: 元の error は `cause` で保持し（`throw new EnrichmentError('...', { cause: error })`）、上位がスタックを失わないようにする
3. **catch した側でハンドリングを完結させる**: 上位に伝えないなら、その時点で適切なレベルでログを出す

「上で catch されているはず」を期待して下位で何もしないのは禁止。
上位で確実にログが出る前提を、コードまたは PR description で示せること。

### フォールバック値で継続する場合

エラー時に代替値を返して処理を継続する設計は **以下を全て満たす場合のみ許容**:

1. 業務的にフォールバックで継続することが明示的に要求されている（仕様 / Issue に記載がある）
2. フォールバック発生を **warn 以上のログで残している**（サイレントにしない）
3. フォールバック頻度を Grafana / Slack で監視可能にしている、または PR で監視追加の必要性を判断している
4. フォールバック値で継続することによる **下流データの整合性** が PR で検証されている

「とりあえず空配列で返す」「とりあえず null で返す」をログなしで行うのは
Critical 違反として扱う。

### レビュー観点（エラーハンドリング）

- `catch` ブロックでログが出ているか、または再 throw されているか
- ログレベルが「型名」ではなく「そのエラーが機能にとって持つ意味」で判定されているか
- `err` フィールドに `serializeError(error)` の戻り値が入っているか（`.message` 単体ではないか）
- `event` / context フィールド（処理対象 ID・ステージ）が構造化されているか
- 必須リソースの欠損が warn 以下に降格されていないか
- `.catch(() => undefined)` / `.catch(() => [])` / `.catch(() => null)` のような握りつぶしが含まれていないか
- `if (!result) return;` で本来 error / fatal レベルの欠損が静かに skip されていないか

---

## レビュー観点（truncate）

- 「他 N 件」「…」「truncated」「省略」が通知 / ログ本文に含まれていないか。
- ループ内で `slice` / `take` / 件数上限が掛かっていないか。
- 通知の件数が増えたとき分割送信されるか、それとも切れるかが PR で検証されているか。
- 切る場合は「物理上限の回避」例外条件に該当することを PR description で明記しているか。

## CI ガード

`scripts/check-observability-truncation.ts` が `...他${N}` / `…他${N}` / `(他 ${N})` /
`他 ${N} 枚` など、単位に依存しない省略 suffix を検出し、`.github/workflows/test.yml`
の guards job で失敗させる。

エラーハンドリングのサイレント catch は `packages/eslint-plugin-graph/` の
`graph/no-silent-catch` ルール（`error` レベル）で機械的に検出する。
`catch (...) {}` / `.catch(() => <literal>)` 等のパターンを検出し、
`@cortex/otel/logger` での構造化ログ出力または再 throw を強制する。

---

## Grafana Loki アラートクエリパターン

`infra/observability/grafana-alert-rules.ts` で Loki アラートルールを追加・変更するときのパターン。

### `|= "error"` テキストフィルタ禁止

Loki クエリで `|= "error"` テキストフィルタを使うと、ログ行に "error" を含む **非エラーログ** が誤検知される。

**禁止例**:

```logql
count_over_time({service_name=~"..."} |= "error" [5m])
```

以下は実際に誤検知が起きたパターン:
- CSS クラス名（`ux-mirror` の `.error-message`）
- TypeScript コンパイラ出力（`"0 errors"` / `"Found 1 error"`）
- 正常完了メッセージに "error" を含む場合（`"total_errors: 0"`）

### `detected_level` ストリームラベルを使う

`@cortex/otel` + Pino + OpenTelemetry SDK を使う全サービスでは、Grafana Cloud の OTel ingest パイプラインが `severity_number` から `detected_level` をインデックス付き **ストリームラベル** として付与する（構造化メタデータではない）。

**推奨: ストリームセレクター内で使用（インデックス効率が良い）**:

```logql
count_over_time({service_name=~"...", detected_level="error"} [5m])
```

**代替: パイプラインフィルターでも動作する**:

```logql
count_over_time({service_name=~"..."} | detected_level="error" [5m])
```

`detected_level` がストリームラベルであるため、`{..., detected_level="error"}` のストリームセレクター構文が有効であり、インデックスを使った高速なフィルタリングが可能。`| detected_level` のパイプラインフィルター形式もストリームラベルに対して動作するが、`errorSeverityCount` ではログメッセージが "error" を含まない "Logic GCF failed" 系の検知に使用している（どちらの形式も有効）。

### テスト: `toMatchInlineSnapshot` で LogQL クエリ全体をスナップショット固定

アラートクエリを生成するヘルパー関数には `toMatchInlineSnapshot` で期待するクエリ文字列全体を固定し、将来の誤変更を CI で検知できるようにする。`toContain` / `not.toContain` の部分一致アサーションのみは禁止（`docs/guidelines/testing.md` 参照）。

## 参照先

- 重要度判定: [severity.md](./severity.md)
- 再発防止: [recurrence-prevention.md](./recurrence-prevention.md)
- AI アンチパターン（フォールバック濫用）: [ai-antipattern.md](./ai-antipattern.md)
- セキュリティ境界での fail-fast: [security.md](./security.md)
