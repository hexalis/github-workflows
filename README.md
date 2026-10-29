# github-workflows — reusable workflows de Hexalis

Workflows de GitHub Actions reutilizables para los repos de la organización.
Se llaman con `uses:` desde cada repo; las diferencias se parametrizan con `inputs`,
`vars` (no sensibles) y `secrets` (sensibles).

## Workflows disponibles

| Workflow | Qué hace | Entradas principales |
|---|---|---|
| `python-ci.yml` | Setup Python + caché pip + instalar + tests | `python-version`, `working-directory`, `requirements-file`, `test-command` |
| `docker-ghcr.yml` | Build de imagen y push a GHCR (opcional) | `image-name`, `context`, `dockerfile`, `push`, `tags` |
| `deploy-docker.yml` | SSH al servidor → `docker pull` + recrear contenedor + healthcheck | `host`, `user`, `image`, `container-name`, `ports`, `env-file`, `health-url` |

## Uso típico (repo llamador)

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

  build:
    needs: ci
    permissions:
      contents: read
      packages: write
    uses: hexalis/github-workflows/.github/workflows/docker-ghcr.yml@main
    with:
      image-name: ghcr.io/hexalis/api-processor
      dockerfile: "Dockerfile"
      push: ${{ github.event_name != 'pull_request' }}

  deploy:
    needs: build
    if: github.ref == 'refs/heads/main' && github.event_name != 'pull_request'
    uses: hexalis/github-workflows/.github/workflows/deploy-docker.yml@main
    with:
      host: ${{ vars.DEPLOY_HOST }}
      user: ${{ vars.DEPLOY_USER }}
      image: ghcr.io/hexalis/api-processor:latest
      container-name: nasa-api
      ports: "8000:8000"
      env-file: ${{ vars.DEPLOY_ENV_FILE }}
      health-url: "http://127.0.0.1:8000/health"
    secrets:
      ssh-private-key: ${{ secrets.DEPLOY_SSH_KEY }}
      registry-token: ${{ secrets.GHCR_READ_TOKEN }}
```

## Variables vs. secrets

| Tipo | Ejemplos | Dónde se configuran |
|---|---|---|
| `inputs` | versión de Python, comando de tests, nombre de imagen, puertos | en el YAML del caller (`with:`) |
| `vars` | `DEPLOY_HOST`, `DEPLOY_USER`, `DEPLOY_ENV_FILE` | Settings → Variables del repo/org |
| `secrets` | `DEPLOY_SSH_KEY`, `GHCR_READ_TOKEN` | Settings → Secrets del repo/org |

- **No** pongas claves en `inputs` (quedan visibles en logs).
- Para GHCR privado, el servidor necesita un token de solo lectura (`read:packages`) en
  `GHCR_READ_TOKEN`; si el paquete es público, omite ese secret.
- `known-hosts` (opcional) evita el `ssh-keyscan` y da más seguridad; guárdalo como secret.

## Permisos necesarios

- El **repo llamador** debe permitir usar workflows reutilizables (organización).
- El job de build necesita `permissions: packages: write`.
- La clave SSH debe permitir login al usuario de despliegue y ese usuario debe poder ejecutar
  Docker (grupo `docker` o `use-sudo: true`).

## Versionado

En producción, en lugar de `@main` conviene etiquetar este repo (`v1`, `v1.2.0`) y llamar
`@v1`. Así un cambio en los workflows no rompe a todos los repos a la vez.
