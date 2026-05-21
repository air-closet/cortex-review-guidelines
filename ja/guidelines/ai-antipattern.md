# AI 生成コードのアンチパターン

AI コーディングアシスタントが生成したコードに頻出する、人間が書いたコードではめったに見られない問題パターンを検出するための基準。cortex では PR の多くが AI 生成または AI 補助で作られるため、レビュー時に必ずこの観点を通す。

参考: [nrslib/takt の ai-antipattern-reviewer persona / policy](https://github.com/nrslib/takt/blob/main/builtins/ja/facets/personas/ai-antipattern-reviewer.md) を cortex の文脈に合わせて再構成した。

## 役割の境界

**この観点で見る:**

- AI が行った仮定の妥当性検証
- 幻覚 API・存在しないメソッドの検出
- 既存コードベースのパターンとの整合性
- スコープクリープ / スコープ縮小（タスク要件の取りこぼし）
- デッドコード・未使用コード
- フォールバック・デフォルト引数の濫用
- 不要な後方互換コード

**他のガイドラインに任せる:**

- アーキテクチャ → [architecture.md](./architecture.md)
- セキュリティ → [security.md](./security.md)
- Product Graph 整合性 → [graph-integrity.md](./graph-integrity.md)
- 影響範囲分析 → [impact-analysis.md](./impact-analysis.md)

## 行動姿勢

- AI 生成コードは人間がレビューできる速度より速く生成される。品質ギャップを埋めるのがこの観点の存在意義。
- AI は自信を持って間違える。もっともらしく見えるが動かないコード、技術的には正しいが文脈的に間違った解決策を見抜く。
- 信頼するが検証する。初期検査を通過する微妙な問題を捕捉する。

## チェック観点

各項目に **severity** と **scope** を明記する。auto-reviewer は scope に該当しない PR では当該項目を発火させない。severity の判定基準は [severity.md](./severity.md) を優先する。

### 仮定の検証

AI はしばしば仮定を行う。それを検証する。

| severity | scope | 観点 |
|---|---|---|
| Major | 全 PR | 実装が実際に要求された機能・要件と一致しているか（異なる質問に答えていないか） |
| Major | 全 PR | コードベースの他の場所にない独自パターンを使っていないか |
| Minor | 全 PR | 特定の問題に対して過度に汎用的な解決策を持ち込んでいないか |
| Major | 全 PR | ビジネスルール / ドメイン制約を正しく理解しているか |

検証アプローチ:

1. Product Graph で同領域の既存実装を `search_product_graph_nodes` で検索し、命名・構造の規約を確認する。
2. ドメインルールが docs / コメント / テスト名から読み取れる場合、それと矛盾していないか確認する。
3. 実装が「もっともらしい一般解」になっていて、cortex 固有のドメイン語彙が登場しない場合は要警戒。

### もっともらしいが間違っている検出

| severity | scope | 観点 |
|---|---|---|
| Critical | 全 PR | 幻覚 API（使用しているライブラリ・バージョンに存在しないメソッド）を呼び出していないか |
| Critical | 全 PR | 構文は正しいが意味が間違っている（例: 形式だけ検証してビジネスルールを見落とすバリデーション）コードがないか |
| Major | 全 PR | 学習データ由来の非推奨パターン・古い API を使っていないか |
| Critical | 全 PR | 新パラメータ追加時、呼び出し元から実際に値が渡されているか（配線忘れ） |
| Major | 全 PR | エラーハンドリングが現実的なシナリオを網羅しているか（過小エンジニアリングでないか） |
| Minor | 全 PR | タスクに不要な抽象化レイヤーを追加していないか（過剰エンジニアリング） |

検証アプローチ:

1. このコードは実際に build / test が通るか確認する（`pnpm build`, `pnpm test`）。
2. import している関数・型が `node_modules` 上で実在するか確認する。
3. 新規追加されたオプション / 引数を `options.xxx ?? fallback` のような形で受けている場合、呼び出し元から実際に値が渡っているか grep で確認する。
4. ライブラリ API の引数形式は [external-api-clients.md](./external-api-clients.md) の OpenAI / GCP 既知の罠を参照する。

### コピペパターン検出

AI は同じパターンを、間違いも含めて繰り返す傾向がある。

| severity | scope | 観点 |
|---|---|---|
| Major | 全 PR | 繰り返される危険なパターン（複数の場所で同じ脆弱性 / 同じ罠）が含まれていないか |
| Minor | 全 PR | 同じロジックがファイル間で異なる方法で実装されていないか |
| Minor | 全 PR | 抽象化できるはずの不要なボイラープレートが爆発していないか |

[impact-analysis.md](./impact-analysis.md) のセマンティック検索による類似実装走査と組み合わせる。

### 冗長な条件分岐パターン

AI は条件分岐で同一関数を引数の差異のみで呼び分けるコードを生成しがち。

| severity | scope | 観点 |
|---|---|---|
| Minor | 全 PR | `if (x) f(a, b, c) else f(a, b)` のように引数の有無のみで分岐していないか |
| Minor | 全 PR | `if (x) f(a, {opt: x}) else f(a)` のようにオプション有無で同一関数を呼び分けていないか |
| Minor | 全 PR | `if (x) { f(a, x); return; } f(a);` のような戻り値を使わない冗長な else がないか |

```typescript
// REJECT - 両ブランチが同一関数を呼び出し、第 3 引数の有無のみが異なる
if (options.format !== undefined) {
  await processFile(input, output, { format: options.format });
} else {
  await processFile(input, output);
}

// OK - 三項演算子で統一
const formatOpt = options.format !== undefined ? { format: options.format } : undefined;
await processFile(input, output, formatOpt);
```

### コールバック + 外部変数キャプチャの濫用

AI は戻り値で返せるデータを、コールバック関数と外部変数のキャプチャで実装しがち。

| severity | scope | 観点 |
|---|---|---|
| Minor | 全 PR | `let result; await f(x => { result = x })` のようにコールバックで外部変数に代入していないか |
| Minor | 全 PR | イベントハンドラ経由で同期的に値を取得していないか |
| Minor | 全 PR | `forEach` 内で外部 Map / 配列に push して結果を組み立てていないか（`map` / `reduce` で書けるはず） |

```typescript
// REJECT - コールバックで外部変数をキャプチャ
let selectedMode: string | undefined;
await promptUser(choices, (choice) => {
  selectedMode = choice;
});
return selectedMode;

// OK - 戻り値で受け取る
const selectedMode = await promptUser(choices);
return selectedMode;
```

### レビュー指摘への不適切な対応

AI はレビュー指摘を「修正」する代わりに、テストやドキュメントで「指摘内容を検証する」コードを追加して対応したつもりになることがある。

| severity | scope | 観点 |
|---|---|---|
| Critical | レビュー後の修正コミット | 指摘の対象ファイル・対象行への変更が含まれているか（無関係なリファクタで誤魔化していないか） |
| Major | レビュー後の修正コミット | テスト追加が「修正後の正しい動作」を検証しているか（「指摘内容そのもの」を検証してすり替えていないか） |
| Major | レビュー後の修正コミット | DRY 違反等の指摘に対し、ドキュメントで「意図的」と書いて閉じようとしていないか |

[review-guidelines.md](../review-guidelines.md) の「根本対応の原則」も併読する。

### コンテキスト適合性

このコードはこの特定のプロジェクトに合っているか。

| severity | scope | 観点 |
|---|---|---|
| Minor | 全 PR | 既存コードベースの命名規則に一致しているか |
| Minor | 全 PR | エラーハンドリング・ログ出力・テストスタイルがプロジェクトの既存パターンと一貫しているか |
| Minor | 全 PR | プロジェクト規則からの説明のない逸脱がないか |

確認すべき質問:

- このコードベースに精通した開発者ならこう書くか?
- ここに属しているように感じるか?

### インテグレーションパターンの一貫性

同種の API 接続 / データ取得 / 型定義が、プロジェクト内で異なる方式で実装されていないか確認する。

| severity | scope | 観点 |
|---|---|---|
| Major | 全 PR | 同じ目的のコード（REST 呼び出し、データ取得、型定義）で既存実装と異なる方式を新規導入していないか |
| Major | `apps/`, `packages/` | 共通クライアントが存在するのに直接 `fetch` / `axios` で叩いていないか |

検証アプローチ:

1. 変更差分の API 呼び出し / データ取得 / 型定義の方式を確認する。
2. 同目的の既存コードがどの方式で書かれているか Product Graph / grep で確認する。
3. 共通パッケージ（`packages/domain/*`, `packages/external/*` 等）が存在しないかを優先で確認する。
4. 不整合がある場合、プロジェクトの標準パターンへの統一を指摘する。

### スコープクリープ / スコープ縮小

AI は過剰に提供する傾向と、要件の一部を取りこぼす傾向の両方を持つ。

| severity | scope | 観点 |
|---|---|---|
| Major | 全 PR | 要求されていない機能が追加されていないか |
| Minor | 全 PR | 単一実装のためのインターフェース / 抽象化など、早すぎる抽象化がないか |
| Minor | 全 PR | 必要のない設定可能化（過剰設定）がないか |
| Minor | 全 PR | 求められていない「あると良い」追加（ゴールドプレーティング）がないか |
| Major | 全 PR | 明示的な指示がないのに Legacy 値マッピング・正規化ロジックを追加していないか |
| Critical | 全 PR | issue / PR description に書かれた要件のうち実装漏れがないか（スコープ縮小） |

Legacy 対応の判定基準:

- 明示的に「Legacy 値をサポートする」「後方互換性を保つ」という指示がない限り、Legacy 対応は不要。
- `.transform()` による正規化、`LEGACY_*_MAP` のようなマッピング、`@deprecated` な型定義は追加しない。
- 新しい値のみをサポートし、シンプルに保つ。

### 早すぎるキャッシュ戦略の導入

AI はパフォーマンスを「改善」するためにキャッシュ機構を先回りで導入しがち。

| severity | scope | 観点 |
|---|---|---|
| Major | 全 PR | 明示的な要求 / 計測結果なしにキャッシュレイヤー（stale-while-revalidate、インメモリ、Redis、ローカル保存）を追加していないか |
| Minor | 全 PR | ボトルネック未特定のままメモ化を多用していないか |
| Major | 全 PR | TTL・キャッシュキー管理・パージ機構を独自実装で追加していないか |

判断基準: 「キャッシュが必要」という明示的な要求または計測結果があるか。

- YES → 実装してよい
- NO → 実装しない。素朴なデータ取得で十分

### デッドコード検出

AI は新しいコードを追加するが、不要になった旧コードの削除を忘れることが多い。

| severity | scope | 観点 |
|---|---|---|
| Major | 全 PR | リファクタ後に残った未使用の関数・メソッド・変数・定数がないか |
| Major | 全 PR | 早期 return の後の到達不能コード、常に真 / 偽になる条件分岐が残っていないか |
| Major | 全 PR | 呼び出し元の制約により**論理的に到達不能**な防御コードがないか |
| Major | 全 PR | 削除された機能の import 文や package 依存が残っていないか |
| Major | 全 PR | 実体が消えたのに re-export / index 登録だけが残っていないか |
| Minor | 全 PR | コメントアウトされたまま放置されたコードがないか |

論理的デッドコードの例:

```typescript
// REJECT - 呼び出し元がインタラクティブ入力を前提としているため、!isInteractive 分岐は到達不能
function displayResult(data: ResultData): void {
  const isInteractive = process.stdin.isTTY === true;
  const output = isInteractive ? formatRich(data) : formatPlain(data);
}

// OK - 呼び出し元の制約を理解し、不要な分岐を排除
function displayResult(data: ResultData): void {
  logger.info(formatRich(data)); // @cortex/otel/logger を使用
}
```

検証アプローチ:

1. 防御的な分岐を見つけたら、Product Graph / grep で全呼び出し元を確認し、条件を既に満たしていないか確認する。
2. 変更・削除されたコードを参照している箇所がないか grep で確認する。
3. 公開モジュール（index ファイル等）のエクスポート一覧と実体が一致しているか確認する。

### フォールバック・デフォルト引数の濫用

AI は不確実性を隠すためにフォールバックやデフォルト引数を多用する。cortex の [review-guidelines.md](../review-guidelines.md) 「セキュリティと境界」原則とも整合させる。

| severity | scope | 観点 |
|---|---|---|
| Critical | 全 PR | 必須データに対してフォールバックを置いていないか（例: `user?.id ?? 'unknown'`） |
| Major | 全 PR | デフォルト引数を全呼び出し元が省略している（事実上の固定値）状態になっていないか |
| Major | 全 PR | `options?.cwd ?? process.cwd()` のように、上位から値を渡す経路がない null 合体になっていないか |
| Critical | 全 PR | `catch { return ''; }` のように try-catch で本来エラーを空値返却で隠していないか |
| Major | 全 PR | `a ?? b ?? c ?? d` のような多段フォールバックがないか |
| Critical | 全 PR | `if (!x) return;` で本来エラーであるべきケースをサイレントに無視していないか |
| Critical | 設定 / 環境変数 | 必須設定を `baseUrl ?? ''`、`?: string` の optional 化、空文字・同一オリジン・`null` への暗黙フォールバックで隠していないか（型で required にし、起動時 throw で固定する） |
| Critical | 全 PR | `catch` ブロックでエラーログを出さずに継続していないか（`.catch(() => null)` / `.catch(() => [])` / `catch (e) { /* ignore */ }` 等。詳細は [observability.md](./observability.md) の「エラーハンドリングとログレベル」を参照） |
| Major | 全 PR | `logger.error(err.message)` のように `Error.stack` / `cause` を捨てて文字列だけ残していないか（`{ err: serializeError(error) }` で構造化する） |
| Major | 全 PR | `NotFoundError` 等の型名を根拠にログレベルを warn に下げ、業務的に「絶対存在すべきもの」の欠損を握りつぶしていないか |

検証アプローチ:

1. 変更差分で `??` / `||` / `= defaultValue` / `catch` を grep する。
2. 各フォールバック・デフォルト引数について以下を確認する:
   - 必須データか? → REJECT
   - 全呼び出し元が省略しているか? → REJECT
   - 上位から値を渡す経路があるか? なければ REJECT
3. 理由なしのフォールバック・デフォルト引数が 1 つでもあれば REJECT。
4. `catch` を grep し、各ブロックで「ログ出力」「再 throw」「業務的に承認されたフォールバック」のいずれかが満たされているか確認する。エラーレベル判定は [observability.md](./observability.md) の「ログレベルの判定基準」に従う。

### 未使用コードの検出

AI は「将来の拡張性」「対称性」「念のため」で不要なコードを生成しがち。現時点で呼ばれていないコードは削除する。

| severity | scope | 観点 |
|---|---|---|
| Major | 全 PR | 現在どこからも呼ばれていない public 関数 / メソッドがないか |
| Major | 全 PR | 「対称性のため」に作られたが使われていない setter / getter がないか |
| Major | 全 PR | 将来の拡張のために用意されたインターフェース / オプションがないか |
| Major | 全 PR | export されているが grep で使用箇所が見つからないシンボルがないか |

フレームワークが暗黙的に呼び出すライフサイクルフックは例外。

### 不要な後方互換コード

AI は「後方互換のために」不要なコードを残しがち。cortex は**後方互換性を考慮することを原則禁止**としており、これを徹底する。

削除すべき後方互換コード:

| severity | scope | 観点 |
|---|---|---|
| Major | 全 PR | `@deprecated` 付きで使用箇所が無い実装が残っていないか（即削除） |
| Major | 全 PR | 新 API と旧 API が両方存在し、旧 API に使用箇所がないのに残っていないか |
| Major | 全 PR | 互換のために作られ移行完了済みのラッパーが残っていないか |
| Major | 全 PR | `// TODO: remove after migration` のような先送りコメントが放置されていないか |
| Minor | 全 PR | 後方互換のためだけに複雑化した Proxy / アダプタがないか |

判断基準:

1. 使用箇所があるか? → Product Graph / grep で確認。なければ削除。
2. 新旧両方に使用箇所があるか? → 両方が現在使われているなら、後方互換ではなく並存設計の可能性がある。呼び出し元を確認。
3. 外部に公開しているか? → cortex は npm 公開していないため基本即削除可能。
4. データ移行中等の正当な理由があるか? → 完了後に削除する Issue を起票したうえで残す。

AI が「後方互換のため」と言ったら疑う。本当に必要か確認する。

### 決定トレーサビリティ

AI が下した自明でない選択が説明されているか検証する。

| severity | scope | 観点 |
|---|---|---|
| Minor | 全 PR | 自明でない設計選択（ライブラリ選定・パターン選択・トレードオフ）が PR description / コミットメッセージで説明されているか |
| Minor | 全 PR | 理由が技術的に妥当か |
| Minor | 全 PR | 代替案が検討されているか（少なくとも触れられているか） |
| Minor | 全 PR | 仮定が明示的で合理的か |

## 自動レビュー運用

- このガイドラインは `ai-antipattern-reviewer` sub agent から参照される。
- 検出は **差分単位** で行う。リポジトリ全体スキャンはしない。
- 指摘は対象ファイル・行番号・問題・修正理由・確認方法（grep コマンド / Product Graph クエリ）を含める。
- 同じパターンの指摘は family_tag で重複排除する。

## 関連ガイドライン

- [severity.md](./severity.md) - 重要度分類
- [architecture.md](./architecture.md) - アーキテクチャ違反の判定
- [impact-analysis.md](./impact-analysis.md) - 横展開・修正漏れ検出
- [graph-integrity.md](./graph-integrity.md) - Product Graph 整合性
- [recurrence-prevention.md](./recurrence-prevention.md) - lint 化判断
