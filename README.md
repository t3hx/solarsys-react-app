# solarsys

Interactive 3D solar system: the Sun, the eight planets, Pluto, Ceres, Haumea, Makemake, Eris and the Moon, on real J2000 orbital elements, with a sci-fi HUD, guided tutorial and bilingual interface (EN/FR).

Built with React 19, React Three Fiber and Three.js. Rewrite of [solarsys-app-vue](https://github.com/t3hx/solarsys-app-vue) (Nuxt 3, frozen): the audit of that version and the migration plan live in [`docs/`](docs/).

## Features

- Elliptical orbits computed from NASA/JPL J2000 orbital elements; rotation periods and axial tilts follow the IAU convention (Venus, Uranus and Pluto spin retrograde).
- Full-resolution NASA textures on every body: Earth day/night shader with city lights, animated clouds, ocean roughness from the specular map, Venus atmosphere, Saturn and Uranus rings, Pluto bump mapping.
- Click a body to focus and follow it; tactical top-down view of all orbits; time presets of one hour, one day or one year of simulation per second.
- Info window per body (metrics with Earth ratios, analysis and facts), about window, nine-step tutorial, instant EN/FR switch.
- Everything loads during the preloader and is uploaded to the GPU before the scene starts: no loading stall during use.

## Getting started

Requires Node 22+ and [pnpm](https://pnpm.io).

```sh
pnpm install
pnpm dev
```

## Commands

| Command                                                | Description                                                        |
| ------------------------------------------------------ | ------------------------------------------------------------------ |
| `pnpm dev`                                             | Vite dev server                                                    |
| `pnpm build`                                           | Typecheck (`tsc -b`) then production build to `dist/`              |
| `pnpm preview`                                         | Serve `dist/`                                                      |
| `pnpm lint` / `pnpm lint:fix`                          | ESLint                                                             |
| `pnpm format` / `pnpm format:check`                    | Prettier                                                           |
| `pnpm typecheck`                                       | `tsc -b`                                                           |
| `pnpm test` / `pnpm test:watch` / `pnpm test:coverage` | Vitest unit tests                                                  |
| `pnpm test:e2e`                                        | Playwright end-to-end tests (production build served on port 4177) |
| `pnpm icons`                                           | Regenerate the offline Iconify subset                              |

## Stack

Vite, React 19, TypeScript (strict), Three.js via `@react-three/fiber` and `@react-three/drei`, `@react-three/postprocessing`, Zustand, GSAP, Tailwind CSS v4, react-i18next, zod.

## Architecture

See [`DEV_GUIDE.md`](DEV_GUIDE.md) (French) for the developer guide and [`docs/react-migration-plan.md`](docs/react-migration-plan.md) for the full design history. In short: `src/data` validates and normalizes the dataset (zod), `src/physics` solves Kepler orbits and body orientation, `src/scene` renders everything declaratively with R3F, `src/hud`, `src/windows` and `src/onboarding` are plain React on top, and Zustand stores hold only serializable state — Three objects live in refs and a non-reactive registry.

## Deployment

Trunk-based and fully automatic (see [`docs/deployment.md`](docs/deployment.md)); the pipeline lives in [`t3hx/workflows`](https://github.com/t3hx/workflows). Every pull request runs lint, typecheck, unit tests, build and Playwright e2e. Every merge into `main` additionally computes a semver tag from the Conventional Commits, ships `ghcr.io/t3hx/solarsys-react-app:latest` + `:vX.Y.Z` to production through a Dokploy webhook, checks that production serves that version, and only then publishes a GitHub Release. The runtime image is `nginxinc/nginx-unprivileged` serving the static `dist/` on port 8080; the app needs no runtime secret.

## Data sources

Body data is checked against the [NASA Planetary Fact Sheet](https://nssdc.gsfc.nasa.gov/planetary/factsheet/) and JPL J2000 orbital elements by `src/data/reference.test.ts`. Textures come from NASA imagery (see the about window in the app).
