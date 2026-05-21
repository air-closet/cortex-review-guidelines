# Lint ルール作成・運用ガイドライン

cortex では `eslint.config.*` および preset 内部での `no-restricted-syntax` /
`no-restricted-imports` の直書きを禁止する。同等の意図はカスタム ESLint ルールとして
実装し、`@cortex/eslint-plugin-graph` から提供する。

## なぜ直書きを禁止するか

1. **AST セレクタ文字列の検査が効かない** —
   `CallExpression[callee.property.name='toContain']` のようなセレクタ文字列は
   エディタ・型・コンパイラのいずれにもチェックされず、AST 仕様変更時に
   サイレントに無効化する。
2. **DRY が効かない** — 同じ意図のルールが複数 workspace に重複コピーされ、
   メッセージや selector が drift する。
3. **`eslint-disable` 誘惑が出る** — cortex では `eslint-disable` 自体が禁止
   (AGENTS.md _(cortex 内部参照)_)。設定側が緩いと運用で逸脱しやすい。
4. **テスト不可** — selector 文字列のロジックは vitest でカバーできず、
   コードレビューの観点も曖昧になる。

## 推奨フロー

### Step 1: 既存ルールで足りるか確認

[`packages/eslint-plugin-graph/src/rules/`](../../packages/eslint-plugin-graph/src/rules/)
を確認する。代表的な置き換え対応:

| 用途 | 使うルール |
|---|---|
| 弱い Vitest matcher（`toContain`, `toBeTruthy` 等）の禁止 | `graph/vitest-strong-matchers` |
| barrel `index.ts` を re-export 専用に強制 | `graph/reexport-only-index` |
| `@graph-*` JSDoc タグの検証 | `graph/valid-graph-*`, `graph/require-graph-*` |
| `eslint.config.*` での直書き禁止ルール検出 | `graph/no-restricted-syntax-direct-usage` |

### Step 2: カスタムルールを追加する

新しい禁止意図はカスタム ESLint ルールとして実装する。

1. `packages/eslint-plugin-graph/src/rules/<rule-name>.ts` にルール本体を追加
   - `ESLintUtils.RuleCreator` で生成
   - `meta.messages` に固定メッセージを書き、`messageId` で参照する
   - AST visitor は `@typescript-eslint/utils` の `AST_NODE_TYPES` で型安全に書く
2. `packages/eslint-plugin-graph/src/rules/<rule-name>.test.ts` で
   `@typescript-eslint/rule-tester` の `RuleTester` を使い、valid / invalid の
   両方をカバーする（カバレッジ 90% 以上）
3. `packages/eslint-plugin-graph/src/index.ts` の `rules` map にエントリを追加
4. preset 側（`packages/core-ecosystem-config/eslint/`）で
   `'graph/<rule-name>': 'error'` として参照する
5. `packages/eslint-plugin-graph/src/index.test.ts` の inline snapshot を
   再生成する

### Step 3: 直書き禁止の lint で守る

`graph/no-restricted-syntax-direct-usage` ルールが、ESLint config / preset
内部で `no-restricted-syntax` / `no-restricted-imports` のオブジェクトリテラル
キーが現れた場合に error を出す。新規 `eslint.config.*` で直書きすると
`pnpm lint:eslint` が落ちる。

## 許可リスト運用

このガイドラインの趣旨に対する例外は認めない。禁止したい構文・import が増えた場合は:

1. 既存カスタムルールで表現できるか確認する
2. 足りない場合は新しいカスタムルールを追加する
3. カスタムルール化のコストが妥当でない場合は、禁止対象の設計自体を見直す

ルール無効化コメントによる回避は AGENTS.md の禁止事項に従い、認めない。

## 関連

- [packages/eslint-plugin-graph](../../packages/eslint-plugin-graph/) — カスタムルールの実体
- [packages/core-ecosystem-config/eslint](../../packages/core-ecosystem-config/eslint/) — preset 群
- [docs/guidelines/testing.md](./testing.md) — テスト品質基準
- AGENTS.md _(cortex 内部参照)_ — ルール無効化コメント全面禁止のフィロソフィー
