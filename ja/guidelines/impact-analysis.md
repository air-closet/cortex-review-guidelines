# 影響範囲分析ガイドライン

## 目的

PR の変更差分だけでなく、**変更すべきだったのに変更されていない箇所**を cortex-product-graph MCP で特定する。

## 調査手順

### 1. 変更ノードの特定

PR の変更ファイル・関数を cortex-product-graph で検索し、qualifiedName を取得する。

```text
mcp__cortex-product-graph__search_product_graph_nodes(query: "<変更された関数名>")
```

### 2. グラフ走査（構造的な接続）

各変更ノードから forward/backward 両方向にエッジを辿り、影響を受ける可能性のあるノードを列挙する。

```text
mcp__cortex-product-graph__trace_product_graph_connections(
  start_node: "<qualifiedName>",
  direction: "both",
  max_depth: 3
)
```

### 3. セマンティック検索（機能的な類似性）

変更内容のビジネスコンテキスト（`@graph-business` の内容や変更の意図）を自然言語で記述し、類似機能を持つノードを検索する。グラフ上で直接接続されていなくても、同じパターンや同じ概念を扱うコードを発見できる。

```text
mcp__cortex-product-graph__search_product_graph_nodes(
  query: "<変更内容のビジネス的な説明>",
  search_mode: "semantic"
)
```

活用例:
- 関数のバリデーションロジックを変更 → 同様のバリデーションを行う別関数を発見
- エラーハンドリングのパターンを変更 → 同じパターンを使う他の箇所を発見
- BQ テーブルのスキーマ変更 → 同じテーブルを異なるコンテキストで使う関数を発見
- API レスポンス形式の変更 → 同じデータを消費するフロントエンドコードを発見

### 4. 修正漏れの判定

グラフ走査とセマンティック検索の結果から、PR の変更ファイルに含まれていないものを「修正漏れ候補」とする。

以下のパターンに該当する場合は指摘する:

| パターン | 説明 | 例 |
|----------|------|-----|
| **型・インターフェース変更の未伝播** | 型定義を変更したが、その型を参照する関数が未修正 | `KpiSummary` にフィールド追加 → `formatKpiSummary()` が未対応 |
| **via フィールドの不整合** | BQ/Firestore のカラム追加・リネームに対し、`@graph-connects` の `via:` が未更新 | テーブルにカラム追加 → repository の `via:` が古いまま |
| **shares_topic の片側変更** | Pub/Sub トピックの publisher を変更したが subscriber が未対応 | publisher のメッセージ形式変更 → subscriber が旧形式のまま |
| **documented_by の乖離** | コード変更に対応するドキュメントが更新されていない | 新機能追加 → `docs/` の該当ドキュメント未更新 |
| **Repository → UseCase → Handler の連鎖変更** | 下位レイヤーの戻り値型変更が上位レイヤーに未反映 | repository の戻り値変更 → usecase が旧型のまま |
| **テストの未追従** | 変更された関数のテストファイルが PR に含まれていない | `calculateBugRate()` 変更 → `calculateBugRate.test.ts` 未変更 |
| **類似実装の未追従** | セマンティック検索で発見された同パターンのコードが未修正 | チーム別集計ロジック修正 → 別スタックの類似集計が旧ロジックのまま |

### 5. 指摘しないケース

- 変更ノードから 4 ホップ以上離れた間接的な接続
- `@graph-connects none` のユーティリティ関数（外部接続なし）
- 別スタックかつ別デプロイ単位のノード（影響はあるが同一 PR で修正する必要がない）
- ドキュメント更新のみの PR
- セマンティック検索の類似度が低い（距離 0.4 以上）ノード
