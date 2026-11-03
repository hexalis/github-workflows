# github-workflows — reusable workflows de Hexalis

Workflows de GitHub Actions reutilizables para los repos de la organización.
Se llaman con `uses:` desde cada repo; las diferencias se parametrizan con `inputs`,
`vars` (no sensibles) y `secrets` (sensibles).

## Workflows disponibles

| Workflow | Qué hace | Cuándo usarlo |
|---|---|---|
| `python-ci.yml` | Setup Python + caché pip + instalar + tests | Siempre |
| `node-ci.yml` | Setup Node + instalar + lint/format/typecheck/test/build; E2E y seguridad opcionales | Repos Node/TS |
| `changeset-check.yml` | Exige changeset en PRs que tocan paquetes | Repos con Changesets |
| `changesets-release.yml` | Release estable de Changesets con environment | Paquetes npm con Changesets |
| `changesets-prerelease.yml` | Snapshot beta/staging de Changesets | Paquetes npm con Changesets |
| `pages-static.yml` | Build + deploy a GitHub Pages | Sitios estáticos / Storybook |
| **`deploy-build.yml`** | SSH al VPS → `git fetch/reset` → `docker build` → `docker run` → healthcheck | **Recomendado** si el VPS construye la imagen (sin registry) |
| `docker-ghcr.yml` | Build y push de imagen a GHCR | Alternativa: artefacto inmutable / varias instancias |
| `deploy-docker.yml` | SSH → `docker pull` de una imagen publicada → recrear contenedor | Alternativa junto a `docker-ghcr.yml` |

## Flujo recomendado (build en el VPS)

El repo llamador debe tener un `Dockerfile` y un clon del repo en el servidor
(con una deploy key de solo lectura).

```yaml
name: CI/CD

on:
  push:
    branches: [main, develop, qa]
  pull_request:
    branches: [main, develop, qa]

permissions:
  contents: read

jobs:
  ci:
    uses: hexalis/github-workflows/.github/workflows/python-ci.yml@main
    with:
      python-version: "3.14"
      requirements-file: "backend/requirements.txt"
      test-command: "python -m unittest discover -s tests -v"

  deploy:
    needs: ci
    if: github.event_name != 'pull_request'
    uses: hexalis/github-workflows/.github/workflows/deploy-build.yml@main
    with:
      environment: ${{ github.ref_name == 'main' && 'production' || github.ref_name }}
      branch: ${{ github.ref_name }}
    secrets:
      ssh-private-key: ${{ secrets.DEPLOY_SSH_KEY }}
      known-hosts: ${{ secrets.DEPLOY_KNOWN_HOSTS }}
```

La configuración del servidor vive en el **GitHub Environment** que se pasa en
`environment`. El workflow lee `vars.DEPLOY_*` dentro del job, donde el environment ya está
activo.

## Node CI

```yaml
jobs:
  ci:
    uses: hexalis/github-workflows/.github/workflows/node-ci.yml@main
    with:
      package-manager: pnpm
      install-command: pnpm install --frozen-lockfile
      lint-command: pnpm lint
      format-command: pnpm format:check
      typecheck-command: pnpm typecheck
      test-command: pnpm test
      build-command: pnpm build
```

Para npm cambia `package-manager` y los comandos (`npm ci`, `npm run …`). Los jobs `E2E` y
`Security` solo se ejecutan si pasas `e2e-command` / `security-command` (o
`dependency-review: true`).

## Changesets

```yaml
jobs:
  changeset:
    uses: hexalis/github-workflows/.github/workflows/changeset-check.yml@main

  release:
    uses: hexalis/github-workflows/.github/workflows/changesets-release.yml@main
    with:
      environment: stable
      version-command: pnpm version-packages
      publish-command: pnpm release
      auth-check-command: bash scripts/check-npm-auth.sh
    secrets:
      npm-token: ${{ secrets.NPM_TOKEN }}
```

`changesets-prerelease.yml` recibe `environment`, `dist-tag` y opcionalmente
`build-command`/`auth-check-command`. El environment debe tener `NPM_PUBLISH=true` y el
secreto `NPM_TOKEN`.

## GitHub Pages

```yaml
jobs:
  pages:
    uses: hexalis/github-workflows/.github/workflows/pages-static.yml@main
    with:
      package-manager: pnpm
      install-command: pnpm install --frozen-lockfile
      build-command: pnpm build-storybook
      output-path: apps/storybook/storybook-static
      storybook-avatar-url: ${{ vars.STORYBOOK_AVATAR_URL }}
```

El workflow configura `BASE_PATH`, `STORYBOOK_BASE_PATH` y `SITE_URL` antes del build y
despliega en el environment `github-pages`.

Para un sitio estático sin `package.json` usa `package-manager: none`; se omiten
`pnpm/action-setup`, `setup-node` y la instalación, y solo se ejecuta el build:

```yaml
jobs:
  pages:
    uses: hexalis/github-workflows/.github/workflows/pages-static.yml@main
    with:
      package-manager: none
      build-command: mkdir -p _site && rsync -a --exclude 'scripts/' portal/ _site/ && touch _site/.nojekyll
      output-path: _site
```

## Variables vs. secrets

| Tipo | Ejemplos | Dónde se configura |
|---|---|---|
| `inputs` | `environment`, `branch`, `dockerfile`, `context` | en el YAML del caller (`with:`) |
| `vars` | `DEPLOY_HOST`, `DEPLOY_USER`, `DEPLOY_REMOTE_PATH`, `DEPLOY_CONTAINER`, `DEPLOY_PORTS`, `DEPLOY_ENV_FILE`, `DEPLOY_HEALTH_URL` | GitHub Environments (o repo/org para los comunes) |
| `secrets` | `DEPLOY_SSH_KEY`, `DEPLOY_KNOWN_HOSTS` (opcional) | Settings → Secrets del repo/org o del environment |

- En este modelo **no hay token de GHCR**: el deploy construye desde el repo en el VPS.
- El servidor necesita, además, una **deploy key de solo lectura** para hacer `git fetch`
  (se configura en el repo de GitHub y en `core.sshCommand` del clon). No pasa por Actions.
- `DEPLOY_KNOWN_HOSTS` (opcional) evita el `ssh-keyscan`; recomendado en producción.
- En el modelo con GHCR: el job de build necesita `permissions: packages: write` y el servidor
  un token `read:packages` (`GHCR_READ_TOKEN`).

## Requisitos del servidor (deploy-build)

- **Cuenta de servicio dedicada** (recomendado: `deploy`), sin sudo general; solo acceso a
  Docker (grupo `docker` o `use-sudo: true`). Mantén el usuario personal como admin.
- Docker instalado y el usuario de deploy con acceso (`docker` group o `use-sudo: true`).
- `git` instalado y clon del repo en `remote-path`.
- Deploy key de solo lectura configurada.
- Secretos **fuera del home y del repo**: por ejemplo `/etc/hexalis/nasa-api.env`, owner
  `root:deploy` y modo `640` (el usuario de deploy solo lee). Nunca en `/srv` público.
- `curl` disponible para el healthcheck.

## Versionado

En producción, en lugar de `@main` conviene etiquetar este repo (`v1`, `v1.2.0`) y llamar
`@v1`. Así un cambio en los workflows no rompe a todos los repos a la vez.

## Multiambiente

`deploy-build.yml` recibe **un environment** y lee toda la configuración del servidor desde
las variables de ese environment. El caller tiene un solo job de deploy; no hay tres bloques.

| Rama | Environment | Variables propias |
|---|---|---|
| `develop` | `develop` | `DEPLOY_REMOTE_PATH`, `DEPLOY_CONTAINER`, `DEPLOY_IMAGE_TAG`, `DEPLOY_PORTS`, `DEPLOY_ENV_FILE`, `DEPLOY_HEALTH_URL` |
| `qa` | `qa` | las mismas, con valores de QA |
| `main` | `production` | las mismas, con valores de producción |

Variables comunes (repo u organización): `DEPLOY_HOST`, `DEPLOY_USER`, y opcionalmente
`DEPLOY_SSH_PORT`. Secretos: `DEPLOY_SSH_KEY` (obligatorio) y `DEPLOY_KNOWN_HOSTS`
(opcional; si falta se usa `ssh-keyscan`).

Cada environment puede tener una **deployment branch policy** para que solo su rama pueda
desplegarlo. En `production` se pueden añadir **required reviewers** si el plan de GitHub lo
permite.

- Los puertos deben publicarse en `127.0.0.1` si nginx hace proxy.
- Para saltar tests: variable `SKIP_TESTS=true` en el caller (condición en el job de CI).
- Si el frontend es estático (sin Dockerfile), se puede servir con nginx y un workflow de
  `git pull`/rsync; si tiene Dockerfile (Node + nginx), reutiliza `deploy-build.yml` igual que
  el backend, con sus propias variables de environment.
