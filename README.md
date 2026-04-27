# Nuxt 4 nightly — SSR build fails to externalize Vue peer deps

Minimal reproduction for a Nuxt 4 nightly regression.

The current Nuxt 4 nightly (`nuxt-nightly@latest`, `4.4.3-29618876.1bd346ee`)
fails its SSR build because Vite/Rollup can't resolve `vue` (and, once that's
fixed at the project level, `vue-router`) from Nuxt's own
`dist/app/entry.js` / `dist/pages/runtime/plugins/router.js`.

Nuxt 3 and Nuxt 5 nightlies build the same project fine.

## Reproduce the failure

```bash
pnpm install
pnpm repro:4-nightly
```

Expected: build fails with

```
■ Nuxt build error: Error: [vite]: Rollup failed to resolve import "vue" from
  ".../nuxt-nightly/dist/app/entry.js".
```

If you then `pnpm add vue` and rebuild, the next failure shifts to:

```
■ Nuxt build error: Error: [vite]: Rollup failed to resolve import "vue-router" from
  ".../nuxt-nightly/dist/pages/runtime/plugins/router.js".
```

So multiple peer deps that Nuxt's vite-builder is supposed to externalize
for SSR aren't being externalized on this nightly.

## Confirm it's Nuxt 4 nightly–specific

Reset and try the other nightly tags — both build cleanly:

```bash
pnpm reset && pnpm repro:3-nightly   # ✅ builds
pnpm reset && pnpm repro:5-nightly   # ✅ builds
pnpm reset && pnpm repro:4-nightly   # ❌ fails
```

| Tag                   | Version  | Result    |
| --------------------- | -------- | --------- |
| `nuxt-nightly@3x`     | 3.21.3-… | builds    |
| `nuxt-nightly@latest` | 4.4.3-…  | **fails** |
| `nuxt-nightly@5x`     | 5.0.0-…  | builds    |

Same pnpm-alias install pattern in all three, so it's not a consumer-side
config issue — the regression is in the Nuxt 4 nightly track.

## Workaround

Adding `vue` and `vue-router` as direct project dependencies
(`pnpm add vue vue-router`) makes the build pass — Rollup falls back to the
project-level `node_modules`. But that just hides the underlying SSR
externalization issue in the nightly.

There's a script that demonstrates this:

```bash
pnpm reset && pnpm repro:4-nightly-workaround   # ✅ builds
```

vs. the failing version:

```bash
pnpm reset && pnpm repro:4-nightly              # ❌ fails
```

## Note on the `_prep` script

The `repro:*` scripts call a small `_prep` step that pre-creates
`.nuxt/nuxt.lock` and `node_modules/.cache/nuxt/.nuxt/nuxt.lock`. On a
first build the nightly CLI errors with `ENOENT … nuxt.lock` because it
tries to write the lock without ensuring the directory exists. That's
unrelated to the externalization issue this repro is for; the prep is
just there so the actual error is visible in one shot.

## Environment

- pnpm 10.30.0
- Node 22.20.0
- Vite 7.3.2 (bundled by Nuxt)
- Vue 3.5.33 (bundled by Nuxt)
