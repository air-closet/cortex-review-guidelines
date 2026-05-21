# Package Publish Guideline

cortex monorepo の `packages/*` を **GitHub Packages** (`https://npm.pkg.github.com`, scope `@air-closet`) に **public** publish するための共通ワークフローと運用規約。

## 概要

```text
PR で packages/<name>/package.json の version を bump
        │
        └─→ main merge
              │
              └─→ auto-publish.yml が起動
                  ├─ packages/**/package.json を scan
                  ├─ 各 @air-closet/* (private:false) について
                  │  GHP に当該 version が未公開かを判定
                  └─→ 未公開分を matrix で publish-package.yml に渡す
                      ├─ package.json の name / version / publishConfig 検証
                      ├─ pnpm --filter "@air-closet/<name>..." build
                      └─ pnpm publish --access public --provenance --no-git-checks
```

publish は **main へのマージを唯一のトリガ** とする。tag push や手動 release 操作は不要。`package.json.version` が SoT で、これを上げて main に merge した瞬間に GHP へ自動 publish される。

`workflow_dispatch` で手動再実行も可能（transient な publish 失敗のリカバリ用途。再 scan して未公開分のみ publish される）。

## package.json 規約

```jsonc
{
  "name": "@air-closet/<name>",
  "version": "0.0.1",
  "private": false,
  "type": "module",
  "bin": { "<name>": "dist/bin.js" },
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "exports": {
    ".": { "types": "./dist/index.d.ts", "import": "./dist/index.js" }
  },
  "files": ["dist", "README.md"],
  "publishConfig": {
    "access": "public",
    "registry": "https://npm.pkg.github.com",
    "provenance": true
  },
  "repository": {
    "type": "git",
    "url": "https://github.com/air-closet/cortex.git",
    "directory": "packages/<name>"
  },
  "license": "UNLICENSED",
  "scripts": {
    "build": "tsgo -p tsconfig.build.json && chmod +x dist/bin.js",
    "lint": "oxlint --type-aware src/ && eslint src/",
    "test": "vitest run",
    "test:coverage": "vitest run --coverage",
    "clean": "rm -rf dist"
  },
  "dependencies": {},
  "devDependencies": {
    "@cortex/core-ecosystem-config": "workspace:*",
    "@types/node": "catalog:",
    "typescript": "catalog:",
    "vitest": "catalog:"
  }
}
```

### 鉄則

- **`dependencies` を空に保つ** — workspace 依存 (`workspace:*`) を runtime `dependencies` に入れると、`pnpm publish` で `workspace:` 範囲は具体的 semver に書き換えられるが、その semver は **publish されていない** ため install 時に解決失敗する。Node 24 builtin (`node:fs/promises`, `node:path`, `node:child_process`, `node:util`, `node:readline`, `globalThis.fetch`) と `@types/node` のみで完結させる。
- `@cortex/core-ecosystem-config` のようなビルド preset は **`devDependencies` のみ** で参照する。build 後の `dist/` には何も漏れないので tarball に混入しない。
- `files: ["dist", "README.md"]` の whitelist で同梱物を絞る。`.npmignore` は使わない (whitelist 優先で十分かつ可視性が高い)。

## 新規 package 追加チェックリスト

- [ ] `packages/<name>/package.json` の `name` を `@air-closet/<name>` にする
- [ ] `private: false` / `publishConfig.{access, registry, provenance}` を上記テンプレ通りに設定
- [ ] runtime `dependencies` が空 (workspace deps なし) であることを確認
- [ ] `pnpm --filter @air-closet/<name>... build` がエラーなく通る
- [ ] `pnpm --filter @air-closet/<name> pack` で生成 tarball を `tar tvzf` 等で目視し、`dist/` と `README.md` のみ、workspace 依存が含まれていないこと
- [ ] `pnpm --filter @air-closet/<name> publish --dry-run` が成功する
- [ ] `pnpm --filter @air-closet/<name> test:coverage` で 90% (statements AND branches) を満たす

## Version bump と publish

1. PR で `packages/<name>/package.json` の `version` を更新（semver に従う）
2. PR レビュー / CI 通過後に main へ merge
3. `auto-publish.yml` が自動起動するので `gh run watch` で監視
4. publish 成功後、`https://github.com/air-closet/cortex/pkgs/npm/<name>` で provenance バッジ (Built and signed on GitHub Actions) と新 version を確認

同一 PR で複数 package を bump した場合、matrix で並列 publish される。1 つが失敗しても他は継続する (`fail-fast: false`)。

### publish に失敗したら

1. `gh run view <run-id> --log-failed` で原因確認
2. fix を PR で main に入れる（version bump は不要 — 同じ version で再 publish される）
3. それでも transient な失敗（registry 5xx 等）なら `gh workflow run auto-publish.yml` で手動再実行。`auto-publish.yml` は未公開 version のみ拾うので重複 publish にはならない

## consumer 側の利用方法

`@air-closet` scope の package を install / 実行するには GitHub Packages の認証が必要 (public package でも要 token)。

### 1. GitHub Personal Access Token (PAT) を発行

- 設定: <https://github.com/settings/tokens>
- scope: `read:packages` のみで十分
- classic / fine-grained どちらでも可。fine-grained の場合は repository access に air-closet org を含める

### 2. ローカルの `~/.npmrc` に設定

```ini
@air-closet:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GITHUB_PACKAGES_PAT}
```

### 3. 実行

```bash
bunx @air-closet/cc-analyzer@latest --help
# or
npx @air-closet/cc-analyzer@latest --help
```

repo を跨いで再利用する場合は、各 repo の `.npmrc` (commit せず `.gitignore` に追加) か、CI では secret 経由で `NODE_AUTH_TOKEN` を設定し `actions/setup-node@v6` の `registry-url` を使うのが定石。

## Rollback / Deprecation

| ケース | 対処 |
| --- | --- |
| publish 直後 (~72h) に取り下げたい | GitHub Packages の UI または `npm unpublish @air-closet/<name>@<version> --registry https://npm.pkg.github.com` |
| 72h 経過、互換性問題 | `npm deprecate @air-closet/<name>@<broken-version> "use >=<fixed>"` + patch リリース |
| 致命的脆弱性 | deprecate + 直近 LTS 系統に patch リリース。advisory は GitHub Security Advisory で別管理 |

## Troubleshoot

| 症状 | 原因 | 対処 |
| --- | --- | --- |
| main に merge したのに publish が走らない | `packages/**/package.json` が変更ファイルに含まれていない (paths filter ヒットせず) | `package.json` の version を変更して main に push、もしくは `gh workflow run auto-publish.yml` |
| `Validate package metadata` で fail | `publishConfig.registry` / `private` / `name` が規約通りでない | package.json を規約に合わせて修正 → main に再 merge |
| `pnpm publish` が `403 unauthorized` | `GITHUB_TOKEN` 不足 / `permissions: packages: write` 漏れ | reusable workflow 呼び出し側で `permissions` を明示するか、`secrets: inherit` 確認 |
| provenance バッジが付かない | `id-token: write` 不足、もしくは `--provenance` フラグ漏れ | publish job の `permissions.id-token: write` を確認 |
| install 側で `ERESOLVE` や `404 Not Found` | consumer の `.npmrc` 未設定 / scope の registry mapping なし | `@air-closet:registry=https://npm.pkg.github.com` を `.npmrc` に追記 |
| tarball に workspace 依存が含まれる | runtime `dependencies` に `workspace:*` を入れている | `devDependencies` に移動。`dist/` から外部 import を排除 |

## Files

- 配管: [`.github/workflows/publish-package.yml`](../../.github/workflows/publish-package.yml) (reusable)
- トリガ: [`.github/workflows/auto-publish.yml`](../../.github/workflows/auto-publish.yml) (main push + workflow_dispatch)
- 共通 setup: [`.github/actions/setup/action.yml`](../../.github/actions/setup/action.yml)
