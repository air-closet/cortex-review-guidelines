# 外部 API クライアント整備ガイドライン

cortex 内で外部サービス（freee / Channel Talk / Jobcan / SmartHR 等）の REST API を呼び出す共通パッケージを作成・更新する際の標準を定める。

## 原則

- **OpenAPI spec 駆動** — 手書き interface ではなく、OpenAPI 3.x spec から SDK を自動生成する
- **`@hey-api/openapi-ts` を使用** — cortex 標準の SDK generator。`@hey-api/typescript` + `@hey-api/sdk` + `@hey-api/client-fetch` plugin 構成
- **全エンドポイント記述** — spec には公式ドキュメントに載っている全エンドポイントを記述する。consumer が今使っている分だけに絞るのは禁止
- **生成物はコミット対象** — `src/generated/` は CI 環境で generate し直さず、リポジトリに含める
- **barrel 層で named export** — `src/index.ts` は生成 SDK を import し、consumer が実際に使う関数・型のみ named re-export する

## spec ファイルの配置

OpenAPI spec は **パッケージ直下に `openapi.yml` として 1 ファイル** 配置する。

```text
packages/infra/pipeline/<service>-api/
├── openapi.yml                  ← spec 本体
├── openapi-ts.config.ts         ← generator 設定 (input: './openapi.yml')
├── scripts/
│   └── sync-<service>-openapi.sh ← 公式 spec を fetch → YAML 変換 → generate
├── src/
│   ├── auth/                    ← 認証ヘッダ注入 helper (OAuth/Token等)
│   ├── generated/               ← @hey-api/openapi-ts 生成物 (コミット対象)
│   ├── index.ts                 ← barrel (consumer が使う分のみ named export)
│   ├── index.test.ts
│   └── openapi.test.ts          ← spec メタ検証 + path/method 網羅チェック
├── package.json
├── tsconfig.json
└── vitest.config.ts
```

`openapi/` サブディレクトリや `*.swagger.json` / `api-schema.json` 等の命名は使わない。

### YAML を採用する理由

- spec が大規模 (数千行) になっても JSON より可読性が高い
- 公式 spec が JSON でも、`yaml` パッケージで変換可能
- diff レビューが容易

## spec ソースの扱い

| 状況 | 対応 |
|------|------|
| 公式が OpenAPI 3.x JSON を公開 | `sync:openapi` で fetch → YAML 変換して `openapi.yml` に書き戻す |
| 公式が OpenAPI 3.x YAML を公開 | `sync:openapi` でそのまま `openapi.yml` に書き戻す |
| 公式が Swagger 2.0 のみ公開 | `swagger2openapi` 等で OpenAPI 3 に変換し YAML 化 |
| 公式 spec が無い | **手書き** で `openapi.yml` を整備。公式ドキュメントを出典として全エンドポイント網羅 |

### 手書き spec の場合

- 公式 API リファレンスを出典として記述 (URL を `info.description` に残す)
- **`抜粋禁止`**: 公式に載っている全エンドポイント (GET/POST/PUT/DELETE 全メソッド) を記述する。今回使う分だけに絞らない
- `scripts/validate-openapi.sh` 等で OpenAPI spec 構文検証 + path/method 網羅チェックを行う
- consumer の追加要望ごとに `openapi.yml` を拡張するのではなく、最初から網羅しておく

## 認証 helper

OAuth / API Token 等の認証ヘッダ注入は `src/auth/` 配下に置き、barrel から named export する。
生成 SDK の client 設定 (`Config.auth` / `Config.headers`) に注入する helper を提供する。

## consumer への影響

spec の統一・更新で既存 consumer を壊さないよう、以下を守る:

- 既存 API (手書き) と新 SDK (生成) を **並列公開** し、consumer 移行は別 PR で段階的に行う
- 破壊的な型変更が必要な場合は 1 consumer ずつ移行し、全件移行後に旧 API を削除する
- 生成物は `@hey-api/openapi-ts` の default 命名に従い、consumer が安定した名前で import できるよう barrel 層で別名を付けてよい

## package.json の規約

```json
{
  "scripts": {
    "build": "pnpm exec esbuild src/index.ts --bundle --platform=node --format=esm --outfile=dist/index.js --packages=external",
    "clean": "rm -rf dist src/generated",
    "generate": "pnpm exec openapi-ts",
    "lint": "oxlint --type-aware src/ && eslint src/index.ts src/openapi.test.ts",
    "sync:openapi": "./scripts/sync-<service>-openapi.sh",
    "test": "vitest run --config vitest.config.ts"
  },
  "files": ["dist", "src", "openapi.yml"]
}
```

- `files` に `openapi.yml` を含めて公開物として扱う
- `openapi/` ディレクトリは含めない

## 既存パッケージの不整合

過去に作成されたパッケージで本ガイドラインとズレているものは順次 issue 化して修正する。修正済みは表から削除する。

| パッケージ | 不整合 | issue |
|---|---|---|
| （現時点でなし） | - | - |

## ベンダー API の暗黙仕様

公式 OpenAPI spec や SDK の型シグネチャからは見えないが、ベンダー側の内部ルーティング・パラメータ受理可否に影響する暗黙仕様は本セクションに集約する。本番ジョブ停止につながる暗黙仕様を踏んで修正した場合、必ずここを更新して再発を防ぐこと（[recurrence-prevention.md](./recurrence-prevention.md)）。

### OpenAI `images.edit` — `image` は配列で渡すこと

`images.edit` の `image` パラメータを **単一値** (`File` / `Buffer` / `Readable`) で渡すと、OpenAI 側で旧 DALL-E 2 互換エンドポイントにルーティングされる。`gpt-image-1` 系モデル固有のパラメータ（`quality` / `input_fidelity` 等）は旧エンドポイントでは受理されず、リクエストは `400 Unknown parameter: 'quality'` で拒否される。

**正しい呼び方:** 1 要素であっても必ず **配列** で渡す（`image: [fileLike]`）。配列で渡すと `gpt-image-1` 系の multi-image edit エンドポイントにルーティングされ、上記パラメータが受理される。

参考実装と再発防止テスト:

- 修正コミット: `apps/generator/accessory-image-generator/src/generation/openai-image.ts`
- 再発防止テスト: `apps/generator/accessory-image-generator/src/generation/openai-image.test.ts` の `image は配列として OpenAI に渡す` ケース（`toStrictEqual` で完全一致検証）

### OpenAI `images.edit` — `mimeType` は IANA 標準形に正規化すること

`images.edit` は `toFile()` の `type` オプションに渡された mimeType を内部バリデーションし、**IANA 標準形以外**は `400 Invalid file 'image[0]': unsupported mimetype ('image/jpg')` で拒否する。HTTP `Content-Type` ヘッダ由来の値（CDN / 画像配信サーバーが返すレスポンスをそのまま `downloadBinary` で取得した場合）には IANA 非標準値が混ざりうる:

| 入力（HTTP `Content-Type` 由来） | IANA 標準形 |
|---|---|
| `image/jpg` | `image/jpeg` |
| `image/pjpeg` | `image/jpeg` |
| `IMAGE/JPG`（大文字混在） | `image/jpeg` |

**正しい呼び方:** `toFile()` に渡す前に必ず正規化関数を通すこと（小文字化 + `image/jpg` / `image/pjpeg` → `image/jpeg`）。同じ mimeType をファイル拡張子判定（`.jpg` / `.png`）にも使う場合は、正規化後の値を共有して両者がズレないようにする。

参考実装と再発防止テスト:

- 正規化関数: `normalizeMimeTypeForOpenAi` (`apps/generator/accessory-image-generator/src/generation/openai-image.ts`)
- 適用箇所: `openai-image.ts` の `toFile()` 直前 / `image-generator.ts` の拡張子判定
- 再発防止テスト: `openai-image.test.ts` の `normalizeMimeTypeForOpenAi` describe ブロック（`image/jpg` / `image/pjpeg` / 大文字 / 標準値維持を網羅）と `image/jpg 入力時は image/jpeg に正規化して toFile に渡す` 統合ケース

## 参照

- [`@hey-api/openapi-ts` 公式ドキュメント](https://heyapi.dev/)
- [architecture.md](./architecture.md) — パッケージ配置の全体方針
- [recurrence-prevention.md](./recurrence-prevention.md) — 再発防止アーティファクトの必須要件
