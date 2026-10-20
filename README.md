# rssb-devops

Stand-in for RSSB's **central DevOps repository** (plan v3.2 §2.4, §5.3, D12, D22):
reusable GitHub Actions workflows that every app calls, released as **version tags**.
Apps pin a tag (or SHA), never `@main`, so one change cannot hit every app at once.

| Workflow | Called on | Does |
|---|---|---|
| `node-pr-checks.yaml` | every pull request | Gitleaks · hadolint → Semgrep · Trivy fs · tests → build + Trivy image → `all-checks-passed` |
| `node-delivery.yaml` | push to `develop` / `staging` / `main` | build once (amd64 + arm64), scan, push to GHCR, Cosign sign → write **digests** to the GitOps repo: `develop`→uat, `staging`→staging (same image, no rebuild), `main`→prod after approval in the **production** Environment |

## Calling it from an app (≈10 lines)

```yaml
# .github/workflows/delivery.yaml in the app repo
on:
  push:
    branches: [develop, staging, main]
permissions: { contents: read, packages: write, id-token: write }
jobs:
  delivery:
    uses: KAMAHORO123/rssb-devops/.github/workflows/node-delivery.yaml@v1.2.0
    with:
      images: '[{"name":".","context":".","dockerfile":"Dockerfile","target":"runtime","app":"orders-api"}]'
      gitops-repo: KAMAHORO123/platform-gitops
    secrets:
      gitops-deploy-key: ${{ secrets.GITOPS_DEPLOY_KEY }}
```

Secrets are passed explicitly (never `secrets: inherit`). Scanners run from
digest-pinned images; every action is pinned by commit SHA.

## Onboard another app (one command)

```bash
scripts/onboard-app.sh --name orders-api --type nestjs \
  --app-dir ~/code/orders-api --gitops-dir ~/code/platform-gitops [--port 3000] [--database]
```

Copies the hardened Dockerfile (`templates/nestjs|nextjs`), ignore files, scanner
configs, the two caller workflows and Dependabot (7-day cooldown) into the app repo,
and writes `apps/<name>/base` + `envs/{uat,staging,prod}` into the GitOps repo.
Never overwrites existing files.
