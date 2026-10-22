# Déploiement

Référence du processus de mise en ligne. Depuis le 2026-10-22, le projet est **trunk-based** :
une seule branche longue, `main`, et chaque PR mergée part en prod. Le pipeline lui-même vit
dans [`t3hx/workflows`](https://github.com/t3hx/workflows), partagé par tous les projets ;
`.github/workflows/ci.yml` ne fait que l'appeler. Principe de base inchangé : **les builds
ne se font jamais sur le VPS** — la CI teste, construit l'image et la livre.

## Vue d'ensemble

```
PR vers main ─→ verify (lint, format, typecheck, tests, build, e2e)

merge main   ─→ verify ─→ version calculée (à blanc)
             ─→ image ghcr :latest + :vX.Y.Z ─→ webhook ─→ Dokploy PROD
             ─→ vérif /version.json à l'origine ─→ SEULEMENT ALORS : tag git vX.Y.Z + GitHub Release
```

Tout est automatique : on ne déploie jamais à la main. Les branches partent de `main`, une
PR par modification, fusionnée en **squash** : le titre de la PR devient le commit sur `main`
et décide de la version. **Une Release publiée = la prod sert cette version**, par
construction : le tag et la Release ne sont créés qu'après la preuve de mise en ligne.

## Environnement

| Env | Branche | Image | App Dokploy | Secret webhook | Variables |
|---|---|---|---|---|---|
| Prod | `main` | `ghcr.io/t3hx/solarsys-react-app:latest` (+ `vX.Y.Z`, `main-<sha>`) | `solarsys` | `DOKPLOY_PROD_WEBHOOK_URL` | `PROD_URL`, `VPS_TAILSCALE_IP` |

L'app est entièrement statique : **aucun secret runtime**. L'image finale est
`nginxinc/nginx-unprivileged` (non-root) qui sert `dist/` sur le **port 8080**.

L'ancienne app Dokploy `solarsys-preprod` (secret `DOKPLOY_PREPROD_WEBHOOK_URL`, variable
`PREPROD_URL`) n'est plus déployée par la CI. Elle est conservée pour une future preview à
la demande (label `preview` sur une PR).

## Preuve de mise en ligne

Le Dockerfile écrit `dist/version.json` (`{"version":"vX.Y.Z"}`) via le build-arg
`APP_VERSION`. Après le webhook, la CI interroge `version.json` jusqu'à 5 minutes : si la
prod ne sert pas la version attendue, **le job échoue** — un webhook qui répond 200 sans
rien déployer ne passe pas inaperçu.

La vérification se fait **à l'origine, par le tailnet** : `--resolve <host>:80:$VPS_TAILSCALE_IP`
directement sur Traefik — car l'anti-bot de Cloudflare, qui proxifie la prod, répond 403
aux IPs de datacenter des runners GitHub. L'ACL Tailscale de `tag:ci` doit donc autoriser
le VPS sur les ports **443 et 80**.

## Versioning

Automatique, calculé au merge sur `main` depuis les Conventional Commits
(`mathieudutour/github-tag-action`, à blanc d'abord) :

- **première release du repo → `v1.0.0`** ;
- `fix:`, et tout le reste par défaut → **patch** ;
- `feat:` → **minor** ;
- `feat!:` / `BREAKING CHANGE:` → **major**.

L'image prod est taguée `latest` **et** `vX.Y.Z` : on peut épingler ou rollbacker une
version précise dans Dokploy en changeant le tag de l'image. Le tag git et la **GitHub
Release** (notes générées) sont posés en fin de pipeline, après la vérification prod.

## Garde-fous

- `pr-checks.yml` : titre de PR au format Conventional Commits (c'est lui qui fixe la
  version, puisque la PR est fusionnée en squash).
- Ruleset GitHub sur `main` : PR obligatoire, checks `verify / quality`, `verify / e2e` et
  `title` requis, pas de force push ni de suppression. Seul le squash merge est autorisé.
- **Jamais de skip silencieux.** Secret webhook ou variables manquants = job rouge.
- Vérification `/version.json` après chaque déploiement (voir ci-dessus).
- Les runs de PR obsolètes sont annulés (`concurrency`) ; les déploiements sont mis en file,
  jamais interrompus en vol.
- Hook local `~/.claude/hooks/guard-main-push.sh` : tout `git push` vers `main` exige une
  confirmation explicite (pousser `main` = déployer).

## Topologie réseau

Le VPS n'expose rien directement au web public :

- **Apps publiques (prod)** : exposées par le **tunnel Cloudflare** — on ajoute une route
  *Public Hostname* dans Zero Trust → Tunnels, c'est elle qui crée l'entrée DNS proxyfiée
  et Cloudflare termine le TLS.
- **Apps privées (preview, Dokploy, outils)** : enregistrement A manuel (DNS only) vers
  l'**IP Tailscale** du VPS — joignables uniquement depuis le tailnet. Les certificats
  Let's Encrypt sont émis par Traefik via le **challenge DNS** (API Cloudflare), donc pas
  besoin d'être joignable publiquement.
- **Runners GitHub** : Dokploy (webhooks) étant sur le tailnet, le job de deploy rejoint le
  tailnet en **node éphémère** (`tailscale/github-action`, `tag:ci`, client OAuth
  Tailscale → secrets `TS_OAUTH_CLIENT_ID` / `TS_OAUTH_SECRET`).

## Mise en place Dokploy (runbook)

À faire une fois par projet :

1. Le package ghcr est **public** d'office (il hérite de la visibilité du repo — vérifier
   sur la page du package si le repo est privé, sinon PAT `read:packages` dans Dokploy).
2. Dans Dokploy : projet (`solarsys`) → environnement `production` → **Create Service →
   Application** (`solarsys`).
3. Provider **Docker** → image **complète, registre inclus** :
   `ghcr.io/t3hx/solarsys-react-app:latest`. Pas de credentials (package public).
4. DNS : route tunnel Cloudflare (voir la topologie ci-dessus).
5. Onglet **Domains** → host `solarsys.nspace.link`, path `/`, **container port `8080`**,
   HTTPS, Cert `letsencrypt`.
6. Onglet **Deployments** → copier la **Webhook URL** et la poser en secret GitHub :
   `gh secret set DOKPLOY_PROD_WEBHOOK_URL`.
7. Poser les variables pour la vérification :
   `gh variable set PROD_URL --body "https://solarsys.nspace.link"` et
   `gh variable set VPS_TAILSCALE_IP --body "<IP Tailscale du VPS>"`.
8. Une fois par tailnet : tag `tag:ci` dans les ACLs Tailscale + client OAuth (scope
   Auth Keys Write, tag `tag:ci`) → secrets `TS_OAUTH_CLIENT_ID` / `TS_OAUTH_SECRET`.
9. Premier déploiement : bouton **Deploy** dans Dokploy (ensuite le webhook prend le relais).

## Dépannage

- **`deploy / deploy` rouge « not set »** : étapes 6–7 du runbook non faites.
- **`deploy / deploy` rouge « never served »** : le webhook a répondu mais rien n'est en
  ligne — regarder les logs de déploiement dans Dokploy (pull de l'image, démarrage du
  conteneur). Vérifier aussi que l'app n'est pas restée épinglée sur un `vX.Y.Z`.
- **Dokploy ne pull pas l'image** : vérifier que le package ghcr est public et que le tag
  existe (`docker manifest inspect ghcr.io/t3hx/solarsys-react-app:latest`).
- **Rollback prod** : dans Dokploy, remplacer le tag `latest` par le `vX.Y.Z` voulu et
  Deploy. **Remettre `latest` avant de merger le correctif** : sinon le webhook redéploie
  la version épinglée, la vérification échoue et aucune Release n'est publiée.
- **Merge sur `main` sans Release publiée** : le déploiement a échoué (c'est voulu : pas de
  version fantôme). Corriger, re-merger ou relancer le run — la version recalculée sera la
  même tant que le tag n'existe pas.
- **Modifier le pipeline** : ça se passe dans `t3hx/workflows`, jamais dans ce repo.
