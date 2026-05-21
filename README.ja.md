# cortex レビューガイドライン

[airCloset, Inc.](https://corp.air-closet.com/) で開発している AI-first 開発プラットフォーム [cortex](https://ryantsuji.dev/about) で実際に使われているコードレビューガイドラインです。人間のレビュアーと **Auto Review** AI エージェントの両方が同じガイドラインを参照します。

> **注記**: 本リポジトリで言及する「cortex」は airCloset 社内で独自開発した AI プラットフォームの内部コードネームです。Snowflake Cortex や Palo Alto Networks Cortex 等の既存商用サービスとは**一切関係ありません**。

## このリポジトリは何か

このリポジトリは cortex のproduction PRレビューで実際に使われているガイドラインの **snapshot** です。すべての PR は ── AI 生成だろうが人間が書いたものだろうが ── まず AI (`Auto Review`) によってこれらのガイドラインに照らしてレビューされ、AI 単独で判定できないケースのみ人間のレビューが入ります。

ガイドラインの設計方針:

- **AI レビュアーが hard contract として使う** — 各ファイルが専用 sub-agent (`arch-reviewer` / `graph-reviewer` / `security-reviewer` 等) の context として読み込まれる
- **重要度で grade 分け** — Critical / Major / Minor / Nit、明示的な **降格禁止ルール** あり
- **AI アンチパターン対応** — AI 生成コード特有の罠（幻覚 API、フォールバック濫用、デッドコード、スコープクリープ、早すぎる抽象化）を明示的に検知
- **再発防止 driven** — すべてのバグ修正で判定（lint 化 / 横展開 / guideline 追加 / 何もしない）を PR に記録

これは記事連載 [AIハーネスの心臓部](https://ryantsuji.dev/series/building-ai-harness) ── 特に **Part 3: Auto Review** ── の source です。記事ではレビューパイプライン全体を分解します。

## 構成

```
.
├── ja/                            # 日本語（オリジナル）
│   ├── review-guidelines.md       # エントリーポイント
│   └── guidelines/                # 観点別ガイドライン
│       ├── README.md              # ガイドライン index
│       ├── architecture.md        # Composable Architecture
│       ├── graph-integrity.md     # Product Graph / @graph-* タグ
│       ├── security.md            # 認証・入力検証・機密情報
│       ├── gcp-sdk-usage.md       # Cloud Run + OTel 制約
│       ├── testing.md             # テスト設計・コード品質・命名
│       ├── observability.md       # ログ / Slack / アラート（truncate 禁止）
│       ├── ai-antipattern.md      # ★ AI 生成コードのアンチパターン
│       ├── document-writing.md    # ドキュメント配置・管理
│       ├── impact-analysis.md     # 影響範囲分析（Product Graph 利用）
│       ├── recurrence-prevention.md # バグ修正の判定マトリクス
│       ├── severity.md            # ★ Critical/Major/Minor/Nit + 降格禁止
│       ├── external-api-clients.md
│       ├── lint-rules.md          # カスタム ESLint ルール方針
│       ├── internal-member-identifier.md
│       ├── cloud-run-deploy.md    # Cloud Run 既知の罠
│       ├── package-publish.md     # GitHub Packages workflow
│       └── frontend.md            # apps/web UI/UX 基準
└── en/                            # 英訳
    └── （同じ構成）
```

★ 最初に読む推奨: [`severity.md`](./ja/guidelines/severity.md) と [`ai-antipattern.md`](./ja/guidelines/ai-antipattern.md)。

## AI がこのガイドラインをどう使うか

各ガイドラインファイルに対応する `Auto Review` sub-agent が紐づいています。PR が open されると:

1. orchestrator agent が diff から関連するガイドラインを特定
2. sub-agent が並列で起動、それぞれ 1 つのガイドラインファイルを context として読み込む
3. 各 sub-agent が **重要度（Critical / Major / Minor / Nit）** タグ付きで指摘を発火
4. 指摘が集約され、Critical / Major が含まれていれば merge ブロック (`REQUEST_CHANGES`)
5. 重要度の降格や ガイドラインの緩和が含まれる場合は **人間レビューを必須化**

全 pipeline (webhook → Product Graph context → sub-agents → findings → auto-fix → re-review → merge → 並列 deploy) は Part 3 の記事で詳しく書きます。

## 他チームでの使い方

1. **エントリーポイントを読む**: [`ja/review-guidelines.md`](./ja/review-guidelines.md)
2. **まず 2 つだけ読む**: [`severity.md`](./ja/guidelines/severity.md) と [`ai-antipattern.md`](./ja/guidelines/ai-antipattern.md) ── 最も再利用しやすい policy が詰まっている
3. **自分の stack に合わせて修正**: snapshot は TypeScript / pnpm / Cloud Run / Pulumi に opinionated。必要に応じて置換
4. **AI レビュアーに結線**: 各 `Auto Review` sub-agent が 1 ガイドラインファイルを読む。Claude Code / Codex CLI 等を使うなら、これらのファイルを system context として指せる

## 言語

- **日本語**（オリジナル）: [`ja/`](./ja/) ── production で使っている SSoT
- **英語**（翻訳）: [`en/`](./en/)

両言語は同期して維持されます。production cortex でガイドラインが更新されたら、両方同時に更新します。

## 関連

- **記事連載**: [AIハーネスの心臓部](https://ryantsuji.dev/series/building-ai-harness)
- **Part 2 (Product Graph)**: [AIハーネスの心臓部 ── AIのAIによるAIのためのナレッジグラフ](https://ryantsuji.dev/posts/cortex-product-graph)
- **AI アンチパターン観点の原点**: [nrslib/takt の ai-antipattern-reviewer persona](https://github.com/nrslib/takt/blob/main/builtins/ja/facets/personas/ai-antipattern-reviewer.md) ── cortex の文脈に合わせて再構成

## License

MIT — fork、改変、自社利用ご自由に。

---

このリポジトリは public snapshot として維持しています。production 側の変更は定期的に反映します。ガイドラインそのものは **opinionated で発展途上** ── 理論的純粋性ではなく実 production の trade-off を反映しています。
