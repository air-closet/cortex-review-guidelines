# Product Graph 整合性

Product Graph は cortex のコアであり、VISION.md _(cortex 内部参照)_ が掲げる **「知識の外部化」** の実体そのもの。コードの What（何をしているか）だけでなく Why（なぜそうなっているか）を構造化し、Graph RAG による確定的なルーティング、影響範囲分析、クロスリポジトリ追跡を支える。**Product Graph の品質 = cortex 全体の知識基盤の品質**。

各項目に **severity** と **scope** を明記する。auto-reviewer は scope に該当しない PR では当該項目を発火させない。

## `@graph-*` JSDoc タグ（必須）

`@graph-*` タグは Product Graph の Single Source of Truth。`apps/`, `packages/`, `infra/` 配下の `@graph-*` 対象ファイルでは、公開/非公開を問わずトップレベル宣言（関数・クラス・変数）を対象に機械的に検証される（ESLint `graph/require-graph-connects` で強制）。

| severity | scope | 観点 |
|---|---|---|
| Critical | `apps/`, `packages/`, `infra/`（cli 除く） | 新規の関数・クラス・変数宣言に `@graph-connects` タグが記述されているか（接続なし → `@graph-connects none`） |
| Critical | `apps/`, `packages/`, `infra/`（cli 除く） | 外部接続が `@graph-connects {target} [{edgeType}] {description}` で明記されているか |
| Critical | `apps/`, `packages/`, `infra/`（cli 除く） | `@graph-business` にビジネスコンテキスト（日本語）が記述されているか — これが Embedding の品質を決定する |
| Critical | `apps/`, `packages/`, `infra/`（cli 除く） | `@graph-stack` と `@graph-domain` がファイルレベルまたは主要関数に設定されているか |

**補足**:
- `graph/require-graph-connects` がトップレベル変数にも `@graph-connects` を要求する。レビューではコメント文言より機械的に検証されるルールを優先する
- `graph/suspicious-graph-connects-none` は `@graph-connects none` を付けた宣言のうち、関数 / 変数初期化子に外部呼び出しパターンがあるケースを検知する

## `apps/cli/*` の例外

`apps/cli/*` はローカル実行専用の workspace package であり、Cloud Run / Worker / Pages のようなデプロイ対象ではない。現行運用では `eslint.config.js` の Product Graph ルール適用対象から外しており、`@graph-*` タグは不要とする。

| severity | scope | 観点 |
|---|---|---|
| Critical | `apps/cli` | `apps/cli/*` を graph ルール対象へ変更する場合、`eslint.config.js` と本ガイドライン、[../review-guidelines.md](../review-guidelines.md)、[architecture.md](./architecture.md) を**同一 PR で**更新しているか |
| Major | `apps/cli` | `apps/cli/*/src/**/*.ts` に `@graph-*` タグを機械的に要求していないか（lint の誤発火を避ける） |
| Minor | `apps/cli` | CLI package の設計意図・運用ルールが `docs/` 側に記述され、Product Graph 対象外であることをレビュアー間で共有できる状態になっているか |

## スタック・ドメイン・エッジの整合性

| severity | scope | 観点 |
|---|---|---|
| Critical | `@graph-stack` 追加時 | 値が `STACKS` に登録済みか（新規追加時はリビルド必要） |
| Critical | `@graph-domain` 追加時 | 既存ドメインのいずれかに該当するか（存在しない値は `@graph-*` 不一致として扱う） |
| Critical | `@graph-connects` を持つ宣言 | エッジタイプ（`calls`, `queries`, `writes_to`, `reads_from`, `triggers`, `publishes`, `references` 等）が実際の接続を正しく表現しているか。allowlist は ../product-graph/README.md _(cortex 内部参照)_ を一次情報として参照する |
| Major | リポジトリをまたぐ接続を持つ宣言 | 境界ノードが適切に宣言されているか |

## ドキュメント整合性（`docs/`）

`docs/` 配下の `.md` ファイルは Document ノードとして Product Graph に取り込まれ、`documented_by` エッジでコードノードと接続される。ドキュメントが存在しないスタックはグラフ上で Document ノードを持たず、semantic search でドキュメントからコードに辿れなくなる。

**ドキュメントの書き方は [document-writing.md](./document-writing.md) に従うこと。**document-writing.md の「何を書かないか」に該当する更新（実装詳細の散文説明等）は、ドキュメント更新の必須要件（下記 Critical）を満たしたことにはならない。また当該更新自体は下表（既存スタック変更）で別途 Critical として指摘する。

> **必須**: 処理の追加・修正を含む全ての PR で、対応するドキュメント（`docs/{category}/{name}.md`）の存在と更新状況を必ずチェックすること。ドキュメント更新のみの PR（コード変更なし）は本チェックの対象外。

### 全コード変更共通

| severity | scope | 観点 |
|---|---|---|
| Critical | コード変更を含む全 PR | 変更対象のスタック / アプリに対応するドキュメントが `docs/{category}/{name}.md` に存在するか（存在しなければ作成必須） |
| Critical | コード変更を含む全 PR | 変更内容のうち document-writing.md が定める「Why」「アーキテクチャ判断」「エントリポイント」に影響するものがドキュメントに反映されているか |

### 新規スタック・新規アプリ

| severity | scope | 観点 |
|---|---|---|
| Major | 新規 app / stack | ドキュメントのタイトル（`#` 見出し）がノード名として適切か |
| Major | 新規 app / stack | 該当カテゴリの `README.md` にリンクが追加されているか |
| Major | 新規 app / stack | `document-writing.md` の「何を書くか」「何を書かないか」に準拠しているか |

### 既存スタック・既存アプリの変更

| severity | scope | 観点 |
|---|---|---|
| Critical | 既存 app / stack 変更 | ドキュメントの記述が実装と乖離していないか（古い情報が残っていないか） |
| Critical | 既存 app / stack 変更 | 現在の実装を散文で説明する更新になっていないか（[document-writing.md](./document-writing.md) の「何を書かないか」参照） |
