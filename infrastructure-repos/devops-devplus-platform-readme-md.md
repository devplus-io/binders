---
id: 57ccb41e-1563-413c-a98b-bc9ab41be904
binder: binder-mvvle32a-1
folder: folder-mvw0u6iq
---
DevOps | devplus-platform/README.md

Source: DevPlusInc/devplus-platform/README.md, branch docs/devops-refresh-2026-10, refreshed 2026-10-04

Refreshed 2026-10-04 against origin/main code (DevPlusInc/devplus-platform, README.md).

# devplus-platform

Deployment orchestration for DevPlus — Tilt local development, Helm chart,
tenant Terraform, and CI deploy workflows.

## Layout

```
Tiltfile                # local dev (run from umbrella parent)
TILT.md                 # detailed Tilt guide
CHANGELOG.md            # release notes, keyed by chart version
k8s/helm/devplus/       # Helm chart (all services)
terraform/tenants/      # per-tenant Terraform configs
docs/                   # runbooks and setup guides
scripts/                # helper scripts
tests/                  # platform black-box test suite (see tests/README.md)
hubspot-app/            # HubSpot app project for the support desk
```

## Releasing

**[docs/RELEASE.md](docs/RELEASE.md)** — the runbook: cut a version, deploy dev →
stage → prod, verify, roll back. [docs/CICD.md](docs/CICD.md) explains how the
pipeline works underneath.

One SemVer version pins **every** service: chart `version` == `appVersion` ==
every image tag. Deploys are deliberate — merging to `main` only refreshes a
service's `:latest`; nothing ships until someone cuts a Release.

## Local development (Tilt)

Run from the **umbrella parent directory** (sibling to all `devplus-*` repos):

```sh
tilt up -f devplus-platform/Tiltfile -- --tenant=<name> --mode=<local|atlas|aks> [--tag=<ver>]
```

| Mode | Description |
|------|-------------|
| `local` | k3d + in-cluster MongoDB + local registry (`localhost:5111`) |
| `atlas` | k3d + MongoDB Atlas (credentials from Azure Key Vault) |
| `aks` | AKS + GHCR push (requires `--tag`) |

See [TILT.md](TILT.md) for prerequisites, auth setup, and port forwards.

Go services with `devplus-shared` (including `oidc-service`) use `Dockerfile.tilt`
(umbrella build context + `go mod replace`). `frontend` and the Python
`agent-service` use their production `Dockerfile`. Tilt does not build or deploy
`devplus-mcp` (the chart's `devplusMcp.enabled` defaults to `false`).

## Helm chart

`k8s/helm/devplus` — deploys all services into a tenant namespace.

| image (GHCR) | K8s Service | Port |
|----------------|-------------|------|
| `backend` | `backend` | 8080 |
| `executor` | `executor-job-dispatcher` (Deployment, no Service) | — |
| `governance-service` | `governance` | 8084 |
| `opentofu-server` | `opentofu-server` | 8082 |
| `oidc-service` | `oidc-service` | 8081 |
| `agent-service` | `agent-service` | 8083 |
| `frontend` | `frontend` | 80 |
| `devplus-mcp` | `devplus-mcp` (only when `devplusMcp.enabled`) | 8085 |

GHCR images: `ghcr.io/devplusinc/<key>` (e.g. `governance-service` image →
`governance` K8s Service). Every image's tag defaults to the chart's
`appVersion`, so one chart version pins all eight.

## CI workflows

| Workflow | Purpose |
|----------|---------|
| `CI` | Helm lint/template + Terraform fmt/validate |
| `Release` | Cut a version: promote images by digest + publish the chart, then optionally deploy |
| `Deploy Tenants` | Deploy an already-published `chart_version` to `dev` \| `stage` \| `prod` |
| `_deploy-environment.yml` | Reusable per-tenant/env deploy matrix (plus the optional post-deploy quality gate) |
| `Platform Test Matrix` | Hermetic platform test suite (`tests/`) on PRs, pushes, nightly and service `service-image` dispatches |
| `Dependency Review` | Fails PRs that add moderate+ severity dependencies |

See [docs/RELEASE.md](docs/RELEASE.md) to ship.

## Tenant Terraform

Multi-tenant setup under `terraform/tenants/`. See
[terraform/README.md](terraform/README.md).

## Go workspace

The umbrella `go.work` (parent directory) links shared + Go services for local
`go build`/`go test`. It is gitignored inside each service repo and not used in
CI/Docker builds.
