# workflows-frontend

Reusable GitHub Actions workflows and composite actions for frontend PR pipelines (pnpm/Node stack).

## Usage

```yaml
# .github/workflows/pull_request.yml
name: Workflow PR

on:
  pull_request:
    branches: [main]

jobs:
  vulnerability-scan:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      actions: read
      security-events: write
    steps:
      - uses: juv-dev/workflows-frontend/.github/actions/vulnerability-scan@main

  supply-chain:
    needs: vulnerability-scan
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: juv-dev/workflows-frontend/.github/actions/supply-chain@main

  load-values:
    needs: supply-chain
    runs-on: ubuntu-latest
    permissions:
      contents: read
    outputs:
      version: ${{ steps.values.outputs.version }}
      name: ${{ steps.values.outputs.name }}
    steps:
      - uses: juv-dev/workflows-frontend/.github/actions/load-values@main
        id: values

  unit-test:
    needs: load-values
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    steps:
      - uses: juv-dev/workflows-frontend/.github/actions/unit-test@main
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}

  code-quality:
    needs: unit-test
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: juv-dev/workflows-frontend/.github/actions/code-quality@main

  validate-tag:
    needs: code-quality
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: juv-dev/workflows-frontend/.github/actions/validate-tag@main
```

## Composite actions (`.github/actions/`)

| Action | Qué hace |
|---|---|
| `vulnerability-scan` | Bearer SAST, OSV-Scanner, TruffleHog (pin por SHA) y licencias, en cadena con fail-fast |
| `supply-chain` | `pnpm audit`, credenciales en `.npmrc`, flags de hardening en `pnpm-workspace.yaml` |
| `load-values` | Expone `version`/`name` de `package.json` como outputs |
| `unit-test` | Corre tests con cobertura, comenta el resultado en el PR y **falla si algún métrico baja del umbral** (`coverage-threshold`, default 80%) |
| `code-quality` | Código duplicado (`jscpd`), dependencias/exports sin uso (`knip`), cobertura de tipos (`type-coverage`) |
| `validate-tag` | Bloquea el merge si el tag `v<version>` de `package.json` ya existe |
| `no-ai-artifacts` | Bloquea el merge si el PR agrega archivos de configuración de IA (`.claude/`, `AGENTS.md`, `.claudeignore`, `.cursor/`, etc.) |

## Workflows reutilizables (`workflow_call`)

| Workflow | Uso |
|---|---|
| `frontend-pr-load-values.yml` | Wrapper `workflow_call` de `load-values` |
| `frontend-pr-unit-test.yml` | Wrapper `workflow_call` de `unit-test`, con `coverage-threshold` configurable |
| `frontend-pr-validate-tag.yml` | Wrapper `workflow_call` de `validate-tag` |
| `frontend-release.yml` | Crea el GitHub Release solo si el tag está en `main` |
| `frontend-security-scan.yml` | Security scan programado (push a `main` + cron semanal) |

## Reglas críticas

- No commitear secretos: usar siempre `${{ secrets.* }}`.
- Pinear actions de terceros por SHA (no `@main`/`@v1`), para evitar cambios no auditados en CI.
- Todo workflow productivo vive en `.github/workflows/` o `.github/actions/`. No mantener duplicados sueltos fuera de esas carpetas.
