# frontend.md

`apps/web/` 配下の React + Vite + TanStack Router フロントエンドに対するレビュー基準。`apps/web/product-innovation-lab/`（`product.air-closet.ai`）を reference 実装とする。

## 認証

### ログイン・ログアウトを必ず備える

すべての protected app は以下を持つ。

- ヘッダ右上にログアウト trigger（icon button でも、display name メニュー内でも可）
- 未認証時の `/login` route（または Edge Auth Worker への full-page redirect）
- 401 received 時の自動 redirect（`X-Edge-Auth-Start` ヘッダ尊重 → fallback `/login`）

Reference: `apps/web/product-innovation-lab/src/hooks/useAuth.ts` の `loginMutation` / `logoutMutation`、`apps/web/product-innovation-lab/src/components/auth/ProtectedRoute.tsx`。

| severity | 観点 |
|---|---|
| Critical | ログアウト UI が常時到達可能か（任意の protected route から 1 クリックで実行できるか） |
| Critical | 未認証ユーザーが protected route に直接アクセスしたとき login 系 URL に redirect されるか |
| Major | 401 + `X-Edge-Auth-Start` を受けたら Edge Auth start URL に遷移する API クライアントになっているか |

## テーマ

### ライト / ダークの 2 テーマを実装し切替可能にする

- ヘッダ右上に切替トグル（Sun / Moon アイコン）
- 選択値は `localStorage` に永続化する（リロード後も維持）
- 初回は `prefers-color-scheme` を尊重する
- CSS 変数（例: `var(--color-bg-base)`）で色を引き、Tailwind の `bg-bg-base` 等の semantic class に閉じる

Reference: `apps/web/repository-console/src/hooks/useTheme.ts`、`apps/web/product-innovation-lab/src/hooks/useTheme.ts`。

| severity | 観点 |
|---|---|
| Major | テーマ切替トグルがグローバルヘッダに存在するか |
| Major | 選択値が `localStorage` に永続化されているか |
| Minor | 初回ロードで `prefers-color-scheme` に追従するか |
| Minor | 色がハードコードされず、CSS 変数 / semantic Tailwind class 経由になっているか |

## サイドバー

### 必ずアイコンのみの状態に最小化できる

- 展開 / アイコンのみ の 2 状態を持つ
- 状態は `localStorage` に永続化（リロード後も維持）
- 最小化中は hover で一時展開できる（業務 UI で頻繁にラベルを確認するため）
- モバイル幅では drawer に切り替える（hamburger menu で開閉）

Reference: `apps/web/repository-console/src/hooks/useSidebar.ts` + `apps/web/repository-console/src/app.tsx` の `AppLayout`。

| severity | 観点 |
|---|---|
| Major | サイドバーをアイコンのみに最小化する toggle が実装されているか |
| Major | 最小化状態が `localStorage` に永続化されているか |
| Minor | 最小化中の hover 一時展開 / モバイル drawer が実装されているか |

## ローディング

### Suspense fallback でスケルトンを出す

データフェッチ中の空白画面を出さない。`<Suspense fallback={<XxxSkeleton />}>` でラップし、コンテンツと同形状のスケルトンを表示する。

- スケルトンはコンテンツのレイアウト（カード枠・テーブル行・グラフ高さ）に合わせ、視覚的なジャンプを抑える
- スケルトンは `components/feedback/` 配下に集約（`DashboardSkeleton`、`KpiTableSkeleton` 等の単位で再利用）
- 単発 spinner はトップレベル auth bootstrap など「形が決まらない」場合のみ。それ以外は必ずスケルトン
- `useQuery` の `isLoading` 分岐よりも、`suspense: true` + `<Suspense>` を優先する（コードがフラットになる、ローディング表現を 1 箇所に集約できる）

Reference: `apps/web/product-innovation-lab/src/components/feedback/DashboardSkeleton/`、`KpiTableSkeleton/`、`LoadingScreen/`、`ErrorState/`。

| severity | 観点 |
|---|---|
| Major | データフェッチを伴う route / panel に対応するスケルトンが用意され `<Suspense>` で wrap されているか |
| Major | スケルトンの形状（カード数 / 行数 / 高さ）が本体コンテンツと一致しているか |
| Minor | スケルトン部品が `components/feedback/` に集約されているか |

## ルーティング

### 深いリンクから到達した URL は 404 を出さない

外部通知、メール、GitHub Check Run summary、Slack などから app 内の詳細画面へ直接遷移する deep link は、サイドバーや一覧からの通常導線と同じ品質で扱う。

- 該当 ID が見つからない場合のフォールバック表示（empty state / 一覧に戻すリンク）を必ず描く
- パスパラメータだけで初期表示できるようにし、直前の画面状態や `sessionStorage` に依存しない
- 取得失敗・権限不足・削除済みを区別できる場合は、ユーザーが次に取れる行動に合わせた表示にする

Reference: `apps/web/repository-console/src/routes/SecretlintPage.tsx` の `/secretlint/runs/$runId` deep link。

| severity | 観点 |
|---|---|
| Major | 深いリンク到達時のリソース未存在フォールバックが描かれているか |
| Major | フォールバック状態から一覧・履歴などの復帰導線に戻れるか |
| Major | パスパラメータだけで詳細初期表示に必要なデータ取得が開始されるか |

## レイアウト

### 2ペイン以上のリスト/詳細レイアウトはペイン毎に独立スクロール

リスト + 詳細（master-detail）など複数ペインを横並びにする画面では、各ペインのスクロールを必ず独立させる。ページ全体や `<body>` 側で 1 本のスクロールバーにまとめてはならない。

- ルートコンテナを `flex h-[calc(100vh-<header>-<padding>)] flex-col` などで固定高にし、内部の grid / flex を `min-h-0 flex-1 overflow-hidden` で受ける
- 各ペインは `min-h-0 overflow-y-auto` を直接付与する（`overflow-hidden` の親内部で `min-h-0` を忘れると flex/grid item が縮まずスクロールしない）
- ペイン内のヘッダ・ツールバーは `shrink-0` で固定し、その下の本体だけが `flex-1 overflow-y-auto` でスクロールする
- スクロール位置はペイン間で同期させない

Reference: `apps/web/repository-console/src/routes/SecretlintPage.tsx`。

| severity | 観点 |
|---|---|
| Major | 2ペイン以上のレイアウトでペイン毎の `overflow-y-auto` + `min-h-0` が付与されているか |
| Major | ルートが固定高（`h-[calc(...)]` / `h-screen` 系）で受けられているか |
| Minor | ペイン内ヘッダ/ツールバーが `shrink-0` で固定されているか |

## 表示

### 単一 organization 運用ではリスト表示で org 名を省略

`air-closet/foo` のように全件同じ owner であれば `foo` だけ表示する。組織名プレフィックスを毎行に出すと読みづらくなり、視認性が落ちる。fullName を出すのはコピー&ペースト前提のリンク UI に限る。

| severity | 観点 |
|---|---|
| Minor | 単一 org に閉じたリストで `owner/name` 形式の fullName を出していないか |
