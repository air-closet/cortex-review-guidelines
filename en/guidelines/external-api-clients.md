# External API Client Guideline

Standards for creating and updating shared packages inside cortex that call external services' REST APIs (freee, Channel Talk, Jobcan, SmartHR, etc.).

## Principles

- **OpenAPI spec driven** — generate the SDK automatically from an OpenAPI 3.x spec rather than hand-writing the interface
- **Use `@hey-api/openapi-ts`** — cortex's standard SDK generator. Configured with the `@hey-api/typescript` + `@hey-api/sdk` + `@hey-api/client-fetch` plugin stack
- **Cover every endpoint** — the spec must include every endpoint listed in the official documentation. Narrowing it down to whatever the current consumer happens to use is forbidden
- **Generated output is committed** — `src/generated/` is included in the repository; CI does not regenerate it
- **Named exports through a barrel** — `src/index.ts` imports the generated SDK and re-exports only the functions and types consumers actually use, as named exports

## Spec file layout

Place the OpenAPI spec as **a single file named `openapi.yml` at the package root**.

```text
packages/infra/pipeline/<service>-api/
├── openapi.yml                  ← the spec itself
├── openapi-ts.config.ts         ← generator config (input: './openapi.yml')
├── scripts/
│   └── sync-<service>-openapi.sh ← fetch official spec → convert to YAML → generate
├── src/
│   ├── auth/                    ← auth header injection helpers (OAuth/Token, etc.)
│   ├── generated/               ← @hey-api/openapi-ts output (committed)
│   ├── index.ts                 ← barrel (named exports for what consumers use)
│   ├── index.test.ts
│   └── openapi.test.ts          ← spec metadata + path/method coverage check
├── package.json
├── tsconfig.json
└── vitest.config.ts
```

Do not use an `openapi/` subdirectory or names like `*.swagger.json` / `api-schema.json`.

### Why YAML

- Stays readable even when the spec grows to thousands of lines
- Even when the official spec is JSON, the `yaml` package can convert it
- Easier to diff during review

## Handling the spec source

| Situation | Action |
|------|------|
| Vendor publishes OpenAPI 3.x JSON | `sync:openapi` fetches it, converts to YAML, and writes back to `openapi.yml` |
| Vendor publishes OpenAPI 3.x YAML | `sync:openapi` writes it back to `openapi.yml` as-is |
| Vendor publishes only Swagger 2.0 | Convert to OpenAPI 3 with something like `swagger2openapi` and write out YAML |
| No official spec exists | **Hand-write** `openapi.yml`. Cite the official documentation as the source and cover every endpoint |

### When hand-writing the spec

- Cite the official API reference as the source (keep the URL in `info.description`)
- **`No excerpting`:** include every endpoint (all GET/POST/PUT/DELETE methods) that appears in the official documentation. Do not narrow it down to what you need this time
- Use `scripts/validate-openapi.sh` (or similar) to validate OpenAPI syntax and check path/method coverage
- Cover the entire surface up front rather than extending `openapi.yml` consumer request by consumer request

## Auth helpers

Place OAuth / API token header injection under `src/auth/` and re-export it from the barrel as named exports. Provide helpers that plug into the generated SDK's client configuration (`Config.auth` / `Config.headers`).

## Impact on consumers

To unify and update specs without breaking existing consumers:

- Publish the existing (hand-written) API and the new (generated) SDK **side by side**, and migrate consumers one PR at a time
- When a breaking type change is unavoidable, migrate one consumer at a time and remove the old API only after every consumer has moved
- The generated output follows `@hey-api/openapi-ts`'s default naming. The barrel may rename exports so consumers can import a stable surface

## `package.json` conventions

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

- Include `openapi.yml` in `files` and treat it as part of the published artifact
- Do not include an `openapi/` directory

## Existing packages out of compliance

Packages created before this guideline that no longer match it are tracked as issues and fixed incrementally. Once fixed, they are removed from the table.

| Package | Inconsistency | Issue |
|---|---|---|
| (none at this time) | - | - |

## Implicit vendor API behavior

Implicit vendor behavior — invisible from the official OpenAPI spec and SDK type signatures but affecting the vendor's internal routing or parameter acceptance — is collected in this section. When you hit and fix implicit behavior that can take down a production job, update this section to prevent recurrence ([recurrence-prevention.md](./recurrence-prevention.md)).

### OpenAI `images.edit` — pass `image` as an array

When the `image` parameter of `images.edit` is passed as a **single value** (`File` / `Buffer` / `Readable`), OpenAI internally routes the call to the legacy DALL-E 2 compatible endpoint. The legacy endpoint does not accept parameters specific to the `gpt-image-1` family (`quality` / `input_fidelity` and so on), and the request is rejected with `400 Unknown parameter: 'quality'`.

**Correct call:** always pass an **array**, even for a single element (`image: [fileLike]`). With an array, the request is routed to the `gpt-image-1` multi-image edit endpoint and the parameters above are accepted.

Reference implementation and regression test:

- Fix commit: `apps/generator/accessory-image-generator/src/generation/openai-image.ts`
- Regression test: the `image は配列として OpenAI に渡す` case in `apps/generator/accessory-image-generator/src/generation/openai-image.test.ts` (full-shape verification with `toStrictEqual`)

### OpenAI `images.edit` — normalize `mimeType` to the IANA standard form

`images.edit` validates the `mimeType` passed through `toFile()`'s `type` option internally, and rejects **anything other than the IANA standard form** with `400 Invalid file 'image[0]': unsupported mimetype ('image/jpg')`. Values originating from HTTP `Content-Type` headers (e.g. responses fetched verbatim from a CDN or image server via `downloadBinary`) can contain non-IANA values:

| Input (from HTTP `Content-Type`) | IANA standard form |
|---|---|
| `image/jpg` | `image/jpeg` |
| `image/pjpeg` | `image/jpeg` |
| `IMAGE/JPG` (mixed case) | `image/jpeg` |

**Correct call:** always run the value through a normalization function before passing it to `toFile()` (lowercase + `image/jpg` / `image/pjpeg` → `image/jpeg`). When the same mimeType is also used for file-extension inference (`.jpg` / `.png`), share the normalized value so the two cannot diverge.

Reference implementation and regression test:

- Normalization function: `normalizeMimeTypeForOpenAi` (`apps/generator/accessory-image-generator/src/generation/openai-image.ts`)
- Call sites: immediately before `toFile()` in `openai-image.ts`, and the extension inference in `image-generator.ts`
- Regression test: the `normalizeMimeTypeForOpenAi` describe block in `openai-image.test.ts` (covers `image/jpg` / `image/pjpeg` / mixed case / preservation of standard values) and the `image/jpg 入力時は image/jpeg に正規化して toFile に渡す` integration case

## References

- [`@hey-api/openapi-ts` official documentation](https://heyapi.dev/)
- [architecture.md](./architecture.md) — overall package layout policy
- [recurrence-prevention.md](./recurrence-prevention.md) — required recurrence-prevention artifacts
