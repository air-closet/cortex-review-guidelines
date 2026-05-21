# テスト品質

## 基本原則

### テストは実装詳細ではなく振る舞いを検証する

private メソッドの呼び出し回数や内部データ構造の形を固定すると、仕様が変わっていない
リファクタリングでも落ちやすい。戻り値、状態変化、副作用、公開 API の契約など、
外部から観測できる結果を検証する。

判断基準:

- リファクタリングで落ちるなら壊れやすい
- バグ混入時に落ちるなら価値が高い

### テストポートフォリオは階層で設計する

Test Pyramid を前提に、細かいテストは多く、重いテストは少なく保つ。

- Small / Unit: 速い、局所的、大量
- Medium / Integration: 接続面と契約を検証
- Large / E2E: 主要導線だけを少数に絞る

### flaky を許容しない

「再実行したら通る」はテストの成功ではなく、信頼性低下のシグナルである。赤を信じられ
ない状態になると、CI もレビューも機能しなくなる。

### カバレッジは目的ではなく補助指標

見るべきは数値そのものではなく、重要な業務ルールが守られているか、失敗時に原因を追え
るか、変更時の誤検知が少ないかである。

## テスト設計

### まず失敗モードから考える

実装前に次を明確にする。

1. 何が壊れるとユーザーや業務が困るか
2. その壊れ方をどの層で最も安く検知できるか
3. 1つの不具合をどの層で1回だけ捕まえるか

例:

- 税計算の誤差は Unit
- API 入出力契約の崩れは Integration
- 購入成功フローの断線は E2E

### AAA または Given / When / Then で統一する

全テストを同じ構造で書くと、レビュー速度と保守性が上がる。

- Given / Arrange: 前提データ
- When / Act: 実行
- Then / Assert: 期待結果

### 1テスト1意図に絞る

異なる仕様を1つのテストに混ぜると、失敗時に何が壊れたのか分からない。テスト名は仕様文
として読める形にし、assertion はその仕様を補強する範囲に限定する。

### `index.ts` / barrel テストは機械的に固定する

`index.ts` が単なる re-export の集約点である場合、テストの目的は「何が公開されているか」を
安価に固定することにある。個々の export を `typeof ... === 'function'` や `toBeDefined`
で確認するのは弱く、保守コストも高い。

基本方針:

- `import * as module from './index.js'` で runtime export を一括取得する
- `Object.keys(module).sort()` を `toMatchInlineSnapshot()` で固定する
- snapshot は export 名の公開契約として扱う
- 型 export は runtime に現れないため、`index.ts` テストではなく型テスト側で検証する

例:

```ts
import { describe, expect, it } from 'vitest';
import * as domainModule from './index.js';

describe('barrel exports', () => {
  it('公開APIを集約している', () => {
    expect(Object.keys(domainModule).sort()).toMatchInlineSnapshot(`
      [
        "createItem",
        "deleteItem",
        "getItem",
      ]
    `);
  });
});
```

追加の assertion を足してよいケース:

- `index.ts` 自身がロジックや定数を持つ
- export 名だけでなく、公開値の具体値や挙動も契約として重要
- re-export ではなく、`index.ts` でラップや合成をしている

この場合でも、まずは snapshot で export 一覧を固定し、必要な仕様だけを別テストで足す。
barrel テストの中に実装詳細の確認を混ぜすぎない。

## レイヤー別ガイド

### Unit（Small）

目的は、業務ルールと境界条件を高速に検証すること。

- 純粋関数、ドメインロジック、変換処理を最優先で対象化する
- 空、最小、最大、異常値などの入力境界を明示する
- clock、random、UUID は注入可能にして決定的にする

避けること:

- フレームワーク内部仕様の再テスト
- モック過多で実装詳細しか検証しないこと

### Integration（Medium）

目的は、個々の部品は正しいのに組み合わせで壊れるケースを検知すること。

- API handler + use case、repository + DB などの接続面を検証する
- 外部 I/O 以外はできるだけ本物を使う
- テストデータ生成を共通化して setup コストを下げる

避けること:

- すべてをモックして Unit と同じ価値しかないテストにすること

### E2E（Large）

目的は、本番に近い経路で本当に動くことを保証すること。

- 売上、課金、認証、主要業務フローなどのクリティカル経路に限定する
- ケースを増やしすぎない
- 失敗時にログ、スクリーンショット、トレースなどの観測情報を残す

## モック / スタブの方針

### モックは制御不能または高コストな依存に限定する

妥当な対象:

- 外部 SaaS API
- 課金、メール送信など副作用が重い処理
- 低速な外部システム

### モックで守るべきでないもの

- 実装内部の呼び出し回数だけを検証するテスト
- request / response schema や DB schema を無視したダミー契約

### 境界は契約テストで固める

サービス間通信や公開 API は consumer / provider の契約をテストし、片方だけの変更で壊れ
たことを早期に検知する。

## flaky を防ぐルール

- 時間依存を固定する（fake timer、固定 clock）
- 乱数 seed を固定する
- 共有状態を持たない
- 外部ネットワーク依存を切る
- `sleep` で待たず、条件成立を監視する
- 並列実行前提で衝突しない ID / 資源を使う

## 強いテストを書くための追加技法

### Property-Based Testing

例ベースに加えて、常に成り立つ性質を検証する。人間が想定しない入力組み合わせを探索で
きるため、境界条件の抜け漏れを見つけやすい。

例:

- `sort` 結果が単調増加である
- 入出力で要素の多重集合が保存される

### Mutation Testing

対象コードに小さな変更を入れてテストが落ちるかを見る。重要モジュールだけに限定して運用
すると、見逃している仕様の穴を可視化しやすい。

## 既存の弱いテストを改善する手順

1. 実装詳細に結合したテスト、仕様価値が重複したテスト、失敗時に意味が分からないテストを
   削除候補として洗い出す
2. 仕様ベースで Given / When / Then に書き直す
3. Unit / Integration / E2E の責務を再配分する
4. flaky の根本原因を除去する。retry でごまかさない
5. 必要に応じて、再現率、平均修復時間、mutation score などを補助指標として見る

## 新規 app のカバレッジ達成

新規 app の初期 PR で 90% 達成が困難な場合の正規アプローチ。**閾値の引き下げは禁止**で、テスト可能な構造に分離する方向で達成する。

### I/O 重い app の典型パターン

DB クライアント / 外部 API クライアント / OAuth フロー / Webhook handler は naive な実装ではテストが書けず、結果として 90% 未達になりがち。次の構造で分離する。

1. **業務ロジックを純粋関数に切り出す**: `bigquery-client.ts` の query 結果を整形する処理は別ファイル（`kpi-calculator.ts` 等）に出して unit test を書く。クライアント側は薄い fetch ラッパーに留め、I/O は integration test でカバー
2. **外部 SDK を interface で抽象化**: `interface SecretManager { get(name: string): Promise<string> }` を定義し、production / test で実装を差し替える。SDK 直叩きをテスト対象に含めない
3. **Webhook handler は thin に**: 受信→検証→domain 関数呼び出しの 3 段階に絞る。検証 / domain ロジックは個別関数として unit test、handler 自体は integration test 1 本でカバー

### `// istanbul ignore` の禁止

カバレッジ未達を `istanbul ignore` で逃げるのは禁止。違反は Critical で差し戻す。
ignore が必要に見える行は、その時点で「テスト可能な構造になっていない」シグナルなので、リファクタする。

### 例外: bin / scripts エントリポイント

`bin/{cli-name}.ts` のような bin エントリ（CLI 起動と process.argv 解析だけ）と、
`scripts/*.ts` のような one-shot 運用スクリプトは、coverage 90% を要求しない。これらは
`vitest.config.ts` の `coverage.exclude` で明示的に除外する（個別 ignore コメントは禁止）。

## チーム運用ルール

- テスト失敗は後回しにせず最優先で対処する
- flaky を見つけたら quarantine より root-cause 修正を優先する
- バグ修正時は再発防止テストを同一 PR に含める
- テストヘルパーや fixture は重複したら共通化する
- AI 生成テストは採用前に人間が価値をレビューする

## 弱い Vitest matcher を避ける

「とりあえず通る assertion」でカバレッジだけ稼ぐのを防ぐため、次の matcher は原則避ける。

- `toBeTruthy` / `toBeFalsy`
- `toBeDefined`
- `toBe(true|false)` / `toEqual(true|false)` / `toStrictEqual(true|false)`
- `toContain` / `toContainEqual`
- `expect.any` / `expect.anything` / `expect.objectContaining`

最優先で使うべきなのは、期待値をオブジェクト全体で固定する `toStrictEqual` である。
部分一致や存在確認だけで済ませず、仕様として意味のある出力全体を比較する。

`expect(result.hoge)` のようなプロパティ単位のテストは、意図しない変化が他のプロパティに
入っていても検知できない。基本は `expect(result).toStrictEqual(...)` の形で、戻り値や出力
全体の不変性を検証する。

ただし、テスト対象が大きく `toStrictEqual` の期待値を手で維持しづらい場合は、
`toMatchInlineSnapshot` / `toMatchSnapshot` を使って出力全体の変化を検知してよい。そのうえ
で、業務上重要なプロパティだけを補助的に個別 assertion するのは許容される。

### 動的フィールド（タイムスタンプ・ID 等）の扱い

`toStrictEqual` は完全一致比較なので、`Date.now()` / `crypto.randomUUID()` / DB
auto-increment ID 等の動的値が含まれると flaky になる。`expect.any(Date)` 等で
逃げると、上記の弱い matcher 一覧に該当して NG。次のいずれかで動的値を**固定**してから
`toStrictEqual` する。

1. **clock / random を注入可能にする**: `vi.useFakeTimers()` で時刻を固定、`vi.spyOn`
   で `crypto.randomUUID` を mock する。テスト対象側は `clock: () => Date`、
   `idGenerator: () => string` のような依存注入を受け取れるように設計する
2. **動的フィールドを除外して比較**: `const { createdAt, id, ...rest } = result;
   expect(rest).toStrictEqual(...)` の形で固定。除外した `createdAt` / `id` は
   別 assertion で型 / フォーマットだけ検証する（型確認のみに絞った assertion として `expect(result.createdAt).toBeInstanceOf(Date)` は許容）
3. **snapshot で固定**: `toMatchInlineSnapshot` を使い、動的値を `serializer` で
   normalize する（`expect.addSnapshotSerializer` で UUID → `<UUID>`、Date → `<DATE>`
   に置換）

`expect.any(Date)` / `expect.objectContaining` を採用する場面はレビューで Major
として差し戻す。「動的値が含まれているから」では正当化できない。

代わりに、仕様の意味を持つ具体的な matcher を使う。完全一致を期待するケースでは
`toStrictEqual` を優先する。部分一致でないと保守できない大きな外部 payload では、
重要フィールドを個別 assertion するか、必要な形に map してから `toStrictEqual` で固定する。

- `toStrictEqual`
- `toMatchInlineSnapshot`
- `toMatchSnapshot`
- `toHaveLength`
- `toThrow`
- `toMatch`

### 「含まれていること」テストは原則禁止

部分包含 (containment) で済ませるテストは「**本当に部分包含だけが仕様の本質**」のとき
にしか使ってはならない。`toContain` / `toContainEqual` だけでなく、`String#includes` /
`Array#includes` / 正規表現 `.match()` / `text.split(...).filter(line => line.includes(...))`
のような **「ある文字列が含まれているかどうか」をテスト assertion の根拠にする組み立て**
すべてが対象。

理由は単純で、含まれていることを確認しただけでは「他に何が混ざっているか」「順序は
正しいか」「他の同名トークンが偶然条件を通していないか」が分からない。実装側で
truncate / 重複 / 文字化け / 余計なメタ情報の混入が起きても assertion が通り続けるため、
仕様逸脱を検知できない。

**原則**: 出力全体を `toMatchInlineSnapshot` / `toStrictEqual` で固定する。

- 通知メッセージ・ログ整形・テンプレート出力 → `toMatchInlineSnapshot` で全文固定
- 配列の長さや件数の不変条件 → `toHaveLength` + 中身を `toStrictEqual` で確定
- 「禁止トークンが含まれていないこと」を表現するなら、出力全体を snapshot で固定して
  禁止トークンが現れた瞬間 snapshot が壊れるようにする

**どうしても部分一致が必要な場合**: 同じテストケース内に `toMatchInlineSnapshot` を併設し、
その時点での出力全体を可視化して保存すること。snapshot 単体で「含まれていない/含まれている」
の証跡が読めるなら、補助的な `includes` チェックを追加してもよい。snapshot を併設しない
部分一致 assertion はレビューで Major 差し戻し。

### ESLint プリセットの適用

`@cortex/core-ecosystem-config/eslint` の `createBaseEslintConfig()` がルート
`eslint.config.js` に組み込まれており、すべての workspace は spread 経由で自動的に
`vitest-strong-matchers` preset の適用を受ける。**新規 workspace で個別に preset を
import する必要はない**（単独 import が残っていると Phase B のクリーンアップ対象になる）。

### legacy ignore の禁止

段階移行用の `.lint-legacy/weak-matchers.json` は廃止済み。既存テストも含めて
`vitest-strong-matchers` preset が常に有効で、CI は legacy ignore リストと生成スクリプトの
再導入を拒否する。弱い matcher が必要に見える場合も ignore ではなく、期待値の形を具体化して
テスト本体を直す。

### 適用進捗の可視化

```bash
pnpm lint:vitest-matcher-adoption
pnpm lint:vitest-matcher-adoption -- --ci
```

`--ci` は、テストを持つパッケージで未適用がある場合に exit 1 を返す。ルート
`eslint.config.js` が `createBaseEslintConfig` / preset を組み込んでいる限り、すべての
workspace は root inheritance により adopted と判定される。

## レビュー時のチェック観点

各項目に **severity** と **scope** を明記する。auto-reviewer は scope に該当しない PR では当該項目を発火させない。

### テストの存在

| severity | scope | 観点 |
|---|---|---|
| Critical | テスト対象を持つ全 workspace | カバレッジ 90%（statements AND branches）を満たしているか。閾値の引き下げは禁止（`vitest.config.ts` の `coverage.thresholds` を下げる変更は Critical で差し戻す） |
| Major | コード変更を含む PR | 新規コードにテストが書かれているか（実装と同じディレクトリに `*.test.ts` を配置） |
| Minor | barrel `index.ts` を持つ workspace | re-export のみの `index.ts` にもテストファイルがあるか（pre-commit で 0% チェックに引っかかる） |

### テストの品質

| severity | scope | 観点 |
|---|---|---|
| Major | 全テスト | 実装詳細ではなく、外部から観測できる振る舞いを検証しているか |
| Major | 全テスト | flaky 要因（時間・乱数・共有状態・順序依存）がないか |
| Major | 全テスト | 正常系だけでなく、境界値・エラーケースがカバーされているか |
| Minor | 全テスト | AAA（Arrange-Act-Assert）または Given / When / Then で意図が読める構造か |
| Minor | 全テスト | モックは最小限か（過度なモックは実装詳細への結合） |
| Minor | 全テスト | テスト名が期待される動作を明確に記述しているか |

### エラーハンドリング

| severity | scope | 観点 |
|---|---|---|
| Critical | 全 app | `catch` ブロックが空になっていないか — 必ずログ出力（サイレント失敗の防止） |
| Major | 全 app | エラーメッセージが具体的かつ実用的か |

### ファイルサイズ・構造

| severity | scope | 観点 |
|---|---|---|
| Major | 全 app | 関数の引数が 3 つ以下か（超える場合はオブジェクト引数） |
| Minor | 全 app | 1 ファイルのコード行が 500 行以下か |
| Minor | 全 app | 早期リターンパターンでネストが浅いか |

### 命名・スタイル

| severity | scope | 観点 |
|---|---|---|
| Critical | 全リポジトリ | `eslint-disable` / `oxlint-disable` 禁止（[security.md Lint / 規約](./security.md) が正規定義） |
| Minor | 全リポジトリ | ファイル名: kebab-case、定数: UPPER_SNAKE_CASE、ブール値: is/has/should プレフィックス |
| Nit | 全リポジトリ | コメントは日本語 |
