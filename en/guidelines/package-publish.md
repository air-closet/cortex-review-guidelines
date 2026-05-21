# Package Publish Guideline

The shared workflow and operating rules for publishing `packages/*` from the cortex monorepo to **GitHub Packages** (`https://npm.pkg.github.com`, scope `@air-closet`) as **public** packages.

## Overview

```text
PR bumps the version in packages/<name>/package.json
        │
        └─→ merge to main
              │
              └─→ auto-publish.yml fires
                  ├─ scans packages/**/package.json
                  ├─ for each @air-closet/* (private:false), checks whether
                  │  that version is already published to GHP
                  └─→ passes the unpublished entries to publish-package.yml via a matrix
                      ├─ validates name / version / publishConfig in package.json
                      ├─ pnpm --filter "@air-closet/<name>..." build
                      └─ pnpm publish --access public --provenance --no-git-checks
```

Merging to main is the **sole trigger** for publish. There are no tag pushes or manual release operations. `package.json.version` is the source of truth — bump it and merge to main, and the package is auto-published to GHP.

Manual reruns via `workflow_dispatch` are also available (for recovering from transient publish failures; the rescan still picks up only versions that are not yet published).

## `package.json` conventions

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

### Hard rules

- **Keep `dependencies` empty** — putting a workspace dependency (`workspace:*`) in runtime `dependencies` causes `pnpm publish` to rewrite the `workspace:` range to a concrete semver, but that semver is **not published**, so install will fail to resolve it. Keep yourself confined to Node 24 built-ins (`node:fs/promises`, `node:path`, `node:child_process`, `node:util`, `node:readline`, `globalThis.fetch`) and `@types/node`.
- Build presets like `@cortex/core-ecosystem-config` are referenced **only in `devDependencies`**. They leave no trace in `dist/` after build, so they cannot leak into the tarball.
- Limit shipped artifacts with the `files: ["dist", "README.md"]` whitelist. Do not use `.npmignore` — the whitelist is sufficient and more visible.

## Checklist for adding a new package

- [ ] Set `name` in `packages/<name>/package.json` to `@air-closet/<name>`
- [ ] Set `private: false` and `publishConfig.{access, registry, provenance}` per the template above
- [ ] Confirm runtime `dependencies` is empty (no workspace deps)
- [ ] `pnpm --filter @air-closet/<name>... build` completes without errors
- [ ] Inspect the tarball produced by `pnpm --filter @air-closet/<name> pack` with `tar tvzf` (or similar) and verify it contains only `dist/` and `README.md` — and no workspace dependencies
- [ ] `pnpm --filter @air-closet/<name> publish --dry-run` succeeds
- [ ] `pnpm --filter @air-closet/<name> test:coverage` meets the 90% (statements AND branches) gate

## Version bump and publish

1. In a PR, update `version` in `packages/<name>/package.json` (follow semver)
2. After review / CI passes, merge to main
3. `auto-publish.yml` fires automatically — watch it with `gh run watch`
4. After publish succeeds, verify the provenance badge (Built and signed on GitHub Actions) and the new version at `https://github.com/air-closet/cortex/pkgs/npm/<name>`

When a single PR bumps multiple packages, they are published in parallel via the matrix. A failure in one does not stop the others (`fail-fast: false`).

### When publish fails

1. Check the cause with `gh run view <run-id> --log-failed`
2. Land the fix on main via PR (no version bump needed — the same version will be republished)
3. If the failure is still transient (registry 5xx, etc.), rerun manually with `gh workflow run auto-publish.yml`. `auto-publish.yml` only picks up unpublished versions, so there is no risk of double-publishing

## How consumers use the packages

Installing or running an `@air-closet`-scoped package requires authenticating to GitHub Packages (a token is required even for public packages).

### 1. Issue a GitHub Personal Access Token (PAT)

- Settings: <https://github.com/settings/tokens>
- Scope: `read:packages` is enough
- Either classic or fine-grained is acceptable. For fine-grained, include the air-closet org in repository access

### 2. Configure `~/.npmrc` locally

```ini
@air-closet:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GITHUB_PACKAGES_PAT}
```

### 3. Run

```bash
bunx @air-closet/cc-analyzer@latest --help
# or
npx @air-closet/cc-analyzer@latest --help
```

To reuse across repositories, the standard pattern is either a per-repo `.npmrc` (do not commit it; add it to `.gitignore`) or, in CI, setting `NODE_AUTH_TOKEN` via a secret and using `actions/setup-node@v6`'s `registry-url`.

## Rollback / Deprecation

| Case | Action |
| --- | --- |
| Pull a release immediately (~72h) after publish | Use the GitHub Packages UI or `npm unpublish @air-closet/<name>@<version> --registry https://npm.pkg.github.com` |
| Past 72h, compatibility issue | `npm deprecate @air-closet/<name>@<broken-version> "use >=<fixed>"` + patch release |
| Critical vulnerability | Deprecate + patch the latest LTS line. Advisories are tracked separately via GitHub Security Advisory |

## Troubleshooting

| Symptom | Cause | Action |
| --- | --- | --- |
| Merged to main but publish did not run | `packages/**/package.json` not in the changed files (paths filter did not match) | Change `package.json` version and push to main, or `gh workflow run auto-publish.yml` |
| `Validate package metadata` fails | `publishConfig.registry` / `private` / `name` do not match the convention | Fix `package.json` to match the convention and re-merge to main |
| `pnpm publish` returns `403 unauthorized` | Missing `GITHUB_TOKEN` or missing `permissions: packages: write` | Either declare `permissions` explicitly at the reusable workflow caller, or verify `secrets: inherit` |
| Provenance badge missing | Missing `id-token: write`, or missing `--provenance` flag | Check `permissions.id-token: write` on the publish job |
| Consumer hits `ERESOLVE` or `404 Not Found` on install | Consumer `.npmrc` not configured, or scope-to-registry mapping missing | Add `@air-closet:registry=https://npm.pkg.github.com` to `.npmrc` |
| Workspace dependencies appear in the tarball | A `workspace:*` is declared under runtime `dependencies` | Move it to `devDependencies`. Strip external imports out of `dist/` |

## Files

- Plumbing: [`.github/workflows/publish-package.yml`](../../.github/workflows/publish-package.yml) (reusable)
- Trigger: [`.github/workflows/auto-publish.yml`](../../.github/workflows/auto-publish.yml) (main push + workflow_dispatch)
- Shared setup: [`.github/actions/setup/action.yml`](../../.github/actions/setup/action.yml)
