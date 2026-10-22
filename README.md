# t3hx/workflows

Workflows GitHub Actions réutilisables, partagés par mes projets (solarsys, myPortfolio, owlog).
Les projets les **appellent**, ils ne les copient pas : une correction faite ici profite à tous.

## Utilisation

Le `.github/workflows/ci.yml` d'un projet (pendant un test, remplacer `@v1` par `@main`) :

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

permissions:
  contents: read

jobs:
  verify:
    uses: t3hx/workflows/.github/workflows/verify.yml@v1
    with:
      e2e: true

  deploy:
    if: github.event_name == 'push'
    needs: verify
    uses: t3hx/workflows/.github/workflows/deploy.yml@v1
    # Un workflow appelé ne peut pas avoir plus de droits que son appelant
    permissions:
      contents: write
      packages: write
    with:
      image: ghcr.io/t3hx/<projet>
      prod-url: ${{ vars.PROD_URL }}
      origin-ip: ${{ vars.VPS_TAILSCALE_IP }}
    secrets: inherit
```

## `verify.yml`

Lance, dans cet ordre et seulement s'ils existent dans le `package.json` : `lint`, `format:check`,
`typecheck`, `type-check`, `test`, `build`. Puis, en option, `test:e2e` (Playwright, Chromium).

| Input          | Type    | Défaut | Rôle                                  |
| -------------- | ------- | ------ | ------------------------------------- |
| `node-version` | string  | `"24"` | Version de Node                       |
| `e2e`          | boolean | false  | Lance `pnpm test:e2e` après `quality` |

## `deploy.yml`

Quatre jobs : **version** (semver calculé à blanc depuis les Conventional Commits, `v1.0.0` si
aucun tag) → **image** (GHCR, tags `latest`, `vX.Y.Z`, `main-<sha>`) → **deploy** (rejoint le
tailnet, appelle le webhook Dokploy, attend que `/version.json` serve la nouvelle version à
l'origine) → **release** (tag git + GitHub Release, seulement si la prod sert la version).

| Input       | Requis | Rôle                                                          |
| ----------- | ------ | ------------------------------------------------------------- |
| `image`     | oui    | Image GHCR sans tag, ex. `ghcr.io/t3hx/solarsys-react-app`    |
| `prod-url`  | oui    | URL publique de la prod, ex. `https://solarsys.nspace.link`   |
| `origin-ip` | oui    | IP Tailscale du VPS, pour vérifier sans passer par Cloudflare |

## Contrat d'un projet

1. **Les scripts du `package.json` disent comment.** Un script absent = étape sautée sans erreur
   (`pnpm run --if-present`). Toute particularité d'un projet vit dans ses scripts, jamais ici.
2. **Un `Dockerfile` à la racine écrit `dist/version.json`** à partir du build-arg `APP_VERSION` :

   ```dockerfile
   ARG APP_VERSION=dev
   RUN pnpm build && printf '{"version":"%s"}\n' "$APP_VERSION" > dist/version.json
   ```

3. **Dokploy tire l'image, il ne la construit pas** : provider **Docker**, sur
   `ghcr.io/t3hx/<projet>:latest`.

## Secrets et variables (dans chaque projet)

| Nom                        | Type     | Valeur                                            |
| -------------------------- | -------- | ------------------------------------------------- |
| `TS_OAUTH_CLIENT_ID`       | secret   | Client OAuth Tailscale (`tag:ci`)                 |
| `TS_OAUTH_SECRET`          | secret   | Idem                                              |
| `DOKPLOY_PROD_WEBHOOK_URL` | secret   | Dokploy → app → onglet Deployments → Webhook URL  |
| `PROD_URL`                 | variable | `https://<domaine public>`                        |
| `VPS_TAILSCALE_IP`         | variable | IP Tailscale du VPS                               |

L'ACL Tailscale de `tag:ci` doit autoriser le VPS sur les ports 80 et 443.

## Versions de ce repo

- `@main` : la dernière version, pour tester un changement depuis un projet.
- `@v1` : la version stable sur laquelle les projets sont épinglés. Correction compatible :
  `git tag -f v1 && git push -f origin v1`. Changement qui oblige les projets à modifier leur
  appel (input renommé, secret en plus) : nouveau tag `v2`, et migration projet par projet.

Les montées de version des actions arrivent par Dependabot (mensuel).
