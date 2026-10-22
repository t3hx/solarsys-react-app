# Guide développeur

Guide de travail sur **solarsys** (React 19 + React Three Fiber). L'historique des décisions
est dans [`docs/react-migration-plan.md`](docs/react-migration-plan.md) ; l'audit de la version
Vue d'origine dans [`docs/audit-2026-09.md`](docs/audit-2026-09.md). Ce guide décrit l'état
courant du code.

## Démarrage

```sh
pnpm install
pnpm dev        # serveur Vite
pnpm test       # tests unitaires (Vitest, jsdom)
pnpm test:e2e   # Playwright : build de prod servi sur le port 4177
```

Node 22+ et pnpm. Aucun secret nécessaire en développement : l'app est entièrement statique.

## Arborescence

| Dossier          | Rôle                                                                                                                                                                                                                          |
| ---------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `src/config`     | Constantes : couleurs, échelles, rendu, caméra, zoom. Pas de logique.                                                                                                                                                         |
| `src/data`       | Chargement du JSON (`public/data/solar_system_data.json`), validation zod (`schema.ts`), normalisation en modèle numérique SI (`normalize.ts`), lookups.                                                                      |
| `src/physics`    | Pur et testé : résolution de Kepler (`kepler.ts`), orientation des corps (`rotation.ts`), conversion d'éléments orbitaux entre plans de référence (`orbitalElements.ts`), échelles scène (`scaling.ts`), unités (`units.ts`). |
| `src/store`      | Zustand : `app` (phases preloading → tutorial → scene), `interaction` (survol, sélection, tactique, fenêtres), `simulation` (temps, vitesse), `sceneSpeed` (presets de vitesse par type de scène), `camera`, `debug`.         |
| `src/scene`      | Tout le R3F : corps (`bodies/`), orbites, caméra (`camera/CameraRig`), effets (`effects/`), interaction pointeur, registre non réactif (`registry.tsx`), préchargement des textures.                                          |
| `src/hud`        | Le HUD 2D : cadre, coins, zoom, contrôles de temps et de langue, drawer.                                                                                                                                                      |
| `src/windows`    | Fenêtres info et à propos (`HudWindow` commun).                                                                                                                                                                               |
| `src/onboarding` | Préloader et tutoriel.                                                                                                                                                                                                        |
| `src/i18n`       | react-i18next, locales `en`/`fr`, noms des corps traduits.                                                                                                                                                                    |
| `e2e/`           | Specs Playwright. Les tests unitaires vivent à côté du code en `*.test.ts(x)`.                                                                                                                                                |

## Règles d'architecture

- **Aucun objet Three dans React state ou Zustand.** Les meshes et objets 3D vivent dans des
  refs et dans le registre non réactif (`src/scene/registry.tsx`). Les stores ne contiennent
  que des données sérialisables (ids, flags, nombres).
- Les callbacks `useFrame` lisent `useStore.getState()`, jamais de hook.
- Tout est déclaratif là où R3F le permet : helpers, layers, wireframe sont du JSX et des
  props, pas de `scene.add/remove`.
- Import unique `@/` → `src/`.
- Code, identifiants, strings et logs en **anglais** ; seuls les commentaires peuvent être en
  français. Pas de `console.log` commité (`warn`/`error` seulement, avec parcimonie).

## Fidélité physique

- Les éléments orbitaux sont les valeurs J2000 du JPL ; les données livrées sont verrouillées
  par `src/data/reference.test.ts` contre le NASA Planetary Fact Sheet. **En cas d'écart on
  corrige la donnée, jamais la tolérance du test.**
- Convention IAU : la période de rotation est toujours positive, l'obliquité (0–180°) oriente
  seule le pôle et porte le sens de rotation (au-delà de 90°, rotation apparente rétrograde :
  Vénus, Uranus, Pluton). `rotationDirection` est dérivé, jamais une donnée.
- La lumière solaire n'a pas de décroissance avec la distance (éclairage « vitrine » : un
  terminateur net sur chaque corps).

## Textures et chargement

Les textures sources gardent leur pleine résolution (chaque corps peut être zoomé jusqu'à sa
surface) et il n'y a **pas de compression GPU** (décisions de 2026-09-07). Tout charge pendant
le préloader et `SceneReadyMarker` téléverse chaque texture au GPU avant de déclarer la scène
prête : aucun chargement ni transfert GPU pendant l'utilisation (décision de 2026-09-10).

## API de debug

`window.solarsys` existe en `pnpm dev` et dans les builds faits avec `VITE_EXPOSE_DEBUG_API=1`
(jamais en production) : `select(id)`, `toggleTactical()`, `setSpeed(n)`, `setDistance(d)`,
`rotateCamera(az, pol)`, `stats()`, `scene()`. C'est le levier des tests e2e et des captures.

## Tests

- **TDD** : test qui échoue d'abord, puis le minimum de code, puis refactor au vert. La logique
  pure (`src/physics`, `src/data`, formatters) doit être couverte.
- Playwright build l'app avec `VITE_EXPOSE_DEBUG_API=1` et la sert sur le port 4177. Les
  workers sont plafonnés à 2 : chaque page précharge ~1,9 Go de textures et davantage de pages
  simultanées fait crasher Chromium headless.

## Icônes

Sous-ensemble Iconify embarqué (hors ligne) : ajouter le nom de l'icône dans
`scripts/build-icons.mjs` puis lancer `pnpm icons` pour régénérer
`src/hud/icons.generated.json`.

## Branches et CI/CD

- **Trunk-based** : branches `feat/*`, `fix/*`, `perf/*`, `ci/*`… tirées de `main`, une PR
  par modification, squash merge dans `main`. Conventional Commits, en anglais — le titre
  de PR aussi (vérifié par `pr-checks.yml`) : c'est lui qui devient le commit sur `main` et
  qui fixe la version.
- **Déploiement entièrement automatique**, un seul environnement — la référence complète
  est [`docs/deployment.md`](docs/deployment.md). Le pipeline vit dans
  [`t3hx/workflows`](https://github.com/t3hx/workflows) ; `.github/workflows/ci.yml` ne fait
  que l'appeler :
  - PR vers `main` → verify (lint, format, typecheck, tests, build, e2e) ;
  - merge dans `main` → verify → version calculée → image `:latest` + `:vX.Y.Z` → webhook →
    app Dokploy **prod** → vérification que la prod sert bien la version (`/version.json`)
    → tag git + GitHub Release **seulement après cette preuve**.
    L'image d'exécution est `nginxinc/nginx-unprivileged` et sert `dist/` statique sur le
    port 8080 (non-root), sans aucun secret applicatif.
