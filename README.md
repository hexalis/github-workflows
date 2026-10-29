# github-workflows — reusable workflows de Hexalis

Workflows de GitHub Actions reutilizables para los repos de la organización.
Se llaman con `uses:` desde cada repo; las diferencias se parametrizan con `inputs`,
`vars` (no sensibles) y `secrets` (sensibles).

## Workflows disponibles

| Workflow | Qué hace | Cuándo usarlo |
|---|---|---|
| `python-ci.yml` | Setup Python + caché pip + instalar + tests | Siempre |
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
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:

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
    if: github.ref == 'refs/heads/main' && github.event_name != 'pull_request'
    uses: hexalis/github-workflows/.github/workflows/deploy-build.yml@main
    with:
      host: ${{ vars.DEPLOY_HOST }}
      user: ${{ vars.DEPLOY_USER }}
      remote-path: ${{ vars.DEPLOY_PATH }}
      branch: main
      container-name: nasa-api
      image-tag: nasa-api:latest
      dockerfile: "Dockerfile"
      context: "."
      ports: "8000:8000"
      env-file: ${{ vars.DEPLOY_ENV_FILE }}
      health-url: "http://127.0.0.1:8000/health"
    secrets:
      ssh-private-key: ${{ secrets.DEPLOY_SSH_KEY }}
      known-hosts: ${{ secrets.DEPLOY_KNOWN_HOSTS }}
```

## Variables vs. secrets

| Tipo | Ejemplos | Dónde se configura |
|---|---|---|
| `inputs` | versión Python, comando de tests, puertos, nombre de contenedor | en el YAML del caller (`with:`) |
| `vars` | `DEPLOY_HOST`, `DEPLOY_USER`, `DEPLOY_PATH`, `DEPLOY_ENV_FILE` | Settings → Variables del repo/org |
| `secrets` | `DEPLOY_SSH_KEY`, `DEPLOY_KNOWN_HOSTS` (opcional) | Settings → Secrets del repo/org |

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
- `.env` con secretos fuera del repo (ruta en `env-file`).
- `curl` disponible para el healthcheck.

## Versionado

En producción, en lugar de `@main` conviene etiquetar este repo (`v1`, `v1.2.0`) y llamar
`@v1`. Así un cambio en los workflows no rompe a todos los repos a la vez.

## Multiambiente

El mismo `deploy-build.yml` sirve para `develop`, `qa` y `production`: se llama una vez por
rama con distinto `remote-path`, `container-name`, `image-tag`, `ports` y `env-file`.
Ejemplo de patrón:

```yaml
jobs:
  deploy-develop:
    if: github.ref == 'refs/heads/develop'
    uses: hexalis/github-workflows/.github/workflows/deploy-build.yml@main
    with:
      remote-path: /home/user/app-develop
      branch: develop
      container-name: app-develop
      ports: "127.0.0.1:8001:8000"
      # ...
  # deploy-qa y deploy-prod análogos
```

- Los puertos deben publicarse en `127.0.0.1` si nginx hace proxy.
- Para saltar tests: variable `SKIP_TESTS=true` en el caller (condición en el job de CI).
- Si el frontend es estático (sin Dockerfile), se puede servir con nginx y un workflow de
  `git pull`/rsync; si tiene Dockerfile (Node + nginx), reutiliza `deploy-build.yml` igual que
  el backend.
