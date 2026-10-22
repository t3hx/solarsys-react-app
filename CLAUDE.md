# CLAUDE.md

Guidance for Claude Code when working in this repository.

## Project

**solarsys** is an interactive 3D solar system built with React 19, React Three Fiber and Three.js. It is the rewrite of `t3hx/solarsys-app-vue` (Nuxt 3), which is frozen. The audit of that version and the migration plan are in `docs/` and are the reference for every phase.

Linear project: `Solarsys` (team T3H). EPIC `T3H-182`, features `T3H-183` to `T3H-191`.

## Commands

- `pnpm dev` — Vite dev server
- `pnpm build` — typecheck (`tsc -b`) then production build to `dist/`
- `pnpm preview` — serve `dist/`
- `pnpm lint` / `pnpm lint:fix` — ESLint (flat config, committed)
- `pnpm format` / `pnpm format:check` — Prettier
- `pnpm typecheck` — `tsc -b`
- `pnpm test` / `pnpm test:watch` / `pnpm test:coverage` — Vitest (jsdom)
- `pnpm test:e2e` — Playwright (builds with `VITE_EXPOSE_DEBUG_API=1`, serves `dist/` on port 4177). `window.solarsys` (debug API) exists in `pnpm dev` and in builds made with that flag, never in production builds
- `pnpm icons` — regenerate `src/hud/icons.generated.json` (offline Iconify subset) after adding an icon name to `scripts/build-icons.mjs`

## Stack

Vite 8, React 19, TypeScript 5.9 (strict), Tailwind v4 via `@tailwindcss/vite`, Three 0.185, `@react-three/fiber` 9, `@react-three/drei` 10, `@react-three/postprocessing`, Zustand 5, GSAP, react-i18next, `@iconify/react` with bundled collections, zod, Radix Tabs, `marked` + `dompurify`.

## Conventions

- Single import alias `@/` → `src/`. No `~/`.
- Code, identifiers, strings, logs, commits, PRs and issues in **English**. Code comments may be in French.
- Three.js objects never live in React state or Zustand. They live in refs and in the non-reactive registry (`src/scene/registry.ts`). Stores hold only serializable data (ids, flags, numbers).
- `useFrame` callbacks read `useStore.getState()`, never hooks.
- Everything declarative where R3F allows it: helpers, layers and wireframe are JSX and props, not `scene.add/remove`.
- TDD: write the failing test first, then the minimum code, then refactor with tests green. Pure logic (`src/physics`, `src/data`, formatters) must be covered.
- Physical fidelity: rotation periods are positive and the obliquity (0–180°) alone orients the pole and carries the spin direction (IAU convention; `rotationDirection` is derived). `src/data/reference.test.ts` checks the shipped data against NASA Planetary Fact Sheet / JPL J2000 values: fix the data, never the tolerances. Sun light has no distance decay (showcase lighting, clear terminator on every body).
- No `console.log` in committed code (`warn`/`error` only, and sparingly).
- Never commit, push or merge unless asked. Trunk-based: branches `feat/*`, `fix/*`, `perf/*`, `ci/*`… from `main`; one PR per change, squash merge into `main` (the PR title becomes the commit and drives the version). Merging into `main` deploys to production.

## Layout (target, see `docs/react-migration-plan.md` §3)

`src/config`, `src/data`, `src/physics`, `src/store`, `src/scene`, `src/hud`, `src/windows`, `src/onboarding`, `src/i18n`, `src/styles`. Tests live next to the code as `*.test.ts(x)`. Playwright specs in `e2e/`.

## Textures

Source textures keep their full resolution: every body can be zoomed to its surface. Everything loads during the preloader and `SceneReadyMarker` uploads every texture to the GPU before the scene is declared ready: no loading and no GPU transfer during use (decision of 2026-09-10, after a progressive-loading attempt stalled the first selection). No lossy GPU compression (KTX2) by decision of 2026-09-07. Debug builds expose `window.solarsys.stats()` and `window.solarsys.scene()`.

## Deployment

Reference: `docs/deployment.md`. Trunk-based, one environment, fully automatic. The pipeline lives in `t3hx/workflows` (reusable `verify.yml` + `deploy.yml`); `.github/workflows/ci.yml` only calls it: never edit the pipeline here. Every PR runs verify (lint, format, typecheck, unit, build, e2e). Every merge into `main` → semver tag `vX.Y.Z` from the squashed PR title (Conventional Commits) → image `ghcr.io/t3hx/solarsys-react-app:latest` + `:vX.Y.Z` → Dokploy webhook (`DOKPLOY_PROD_WEBHOOK_URL`) → `/version.json` checked at the origin over the tailnet → only then git tag + GitHub Release. Rollback = pin `:vX.Y.Z` in Dokploy. Runtime image is `nginxinc/nginx-unprivileged` serving `dist/` on port 8080 (non-root). No runtime secrets; the app is static.
