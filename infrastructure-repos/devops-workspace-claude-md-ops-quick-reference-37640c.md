---
id: c8fd7ab9-67b2-408d-ac2d-e65df937640c
binder: binder-mvvle32a-1
folder: folder-mvw0u6iq
---
DevOps | workspace CLAUDE.md (ops quick reference)

Source: devplus-workspace/CLAUDE.md (umbrella, untracked; AGENTS.md is the same file for Codex), refreshed 2026-10-04

Refreshed 2026-10-04 against origin/main code (workspace CLAUDE.md, the ops quick reference; not in git).

# DevPlus Workspace

> Project context for engineers **and** for Claude Code. This file is the source of truth for how the DevPlus distributed workspace fits together and how to run it locally.
>
> **Deep reference lives in [`docs/`](docs/README.md)** — living documentation of the whole platform (own git repo, `DevPlusInc/devplus-docs`): [GLOSSARY](docs/GLOSSARY.md) · [ARCHITECTURE](docs/ARCHITECTURE.md) · [DIAGRAMS](docs/DIAGRAMS.md) · [FLOWS](docs/FLOWS.md) · [DATA_MODELS](docs/DATA_MODELS.md) · [API_SURFACE](docs/API_SURFACE.md) · [SECURITY_MODEL](docs/SECURITY_MODEL.md) · [OPERATIONS](docs/OPERATIONS.md) · [REPO_MAP](docs/REPO_MAP.md) · per-service docs in [docs/services/](docs/services/). **Keep it living**: when a change alters an architecture boundary, data model, API surface, flow, or terminology, update the matching doc in the same session.

## 1. What DevPlus Is

DevPlus is a **multi-tenant, web-based infrastructure-provisioning platform**. Users deploy cloud infrastructure with **Terraform/OpenTofu through a UI — no code** — with **AI agent assistance**. It was migrated from the old `infrasync` monorepo into distributed per-service repos under the **`DevPlusInc`** GitHub org (inside a `devplus` GitHub Enterprise with SAML SSO).

A **tenant** is the core unit of isolation: one tenant maps to one **Kubernetes namespace**, one **MongoDB database**, and one **WorkOS organization**.

### Architecture & Data Flow

```
                          Browser  (<tenant>.localhost:3000  /  <tenant> host in AKS)
                              |
                              v
                    ┌─────────────────────┐     WorkOS AuthKit (SSO / session)
                    │ frontend (React SPA)│ <─────── auth.dev.devplus.io/callback
                    │ nginx :3000 (Svc :80)│
                    └──────────┬──────────┘
                               │  same-origin /api/v1/* (sealed httpOnly cookie)
                               v
                   ┌─────────────────────┐          ┌──────────────────────┐
        ┌──────────│  backend (Go) :8080 │<─────────│ devplus-mcp :8085    │ hosted MCP
        │          └──────────┬──────────┘          │ (OAuth 2.1; optional)│
        │                     │          │          └──────────────────────┘
        v                     v          v
 ┌──────────────┐    ┌────────────────┐     ┌───────────────┐
 │ oidc-service │    │  governance    │     │ agent-service │
 │    :8081     │    │ (Cedar) :8084  │     │  (Py) :8083   │
 │ JWT / JWKS / │    └────────────────┘     │ Azure OpenAI  │
 │ TokenReview  │                            │ (no cloud SDK)│
 └──────┬───────┘                            └───────────────┘
        │  JWKS trusted by AWS / Azure / GCP workload-identity federation
        v
 ┌──────────────────────────┐      watches Mongo queues       ┌──────────────────────┐
 │ executor-job-dispatcher  │────────────────────────────────>│  K8s job pods        │
 │   (Go Deployment,        │   spawns K8s job pods            │  tofu / infracost    │
 │    no HTTP port)         │                                  │  discovery / repo    │
 │                          │                                  │  analysis scans      │
 └────────────┬─────────────┘                                  └──────────┬───────────┘
              │                                                            │
              v                                                           registry /
   ┌────────────────────┐   Terraform Registry Protocol +                 state
   │  opentofu-server   │<──────── HTTP Backend Protocol ─────────────────┘
   │       :8082        │   (modules in GridFS, state in Mongo)
   └─────────┬──────────┘
             │
             v
   ┌───────────────────────────────────────────────────────────────────┐
   │  MongoDB — in-cluster StatefulSet (local) │ Atlas (atlas / aks)    │
   │  database name == tenant name                                      │
   └───────────────────────────────────────────────────────────────────┘
```

All services share the **`devplus-shared`** Go library (models, MongoDB helper, OpenTelemetry bootstrap) and are orchestrated for local dev by **`devplus-platform`** (Tilt + Helm).

> **OpenBao is gone.** Connector secrets are envelope-encrypted in Mongo (`connector_secrets`, key `SECRET_ENCRYPTION_KEY`, required to boot backend and opentofu-server, which also encrypts tofu state at rest). `devplus-mcp` (hosted MCP server, OAuth 2.1 resource server) calls the backend API; the chart renders it only with `devplusMcp.enabled` (chart default false, Terraform `devplus_mcp_enabled` default true) and Tilt never deploys it.

## 2. Repository Layout

The umbrella workspace at **`/Users/helderfeixas/Documents/devplus-workspace`** is a **parent directory (NOT itself a git repo)** that holds every `devplus-*` repo as a sibling, plus an untracked `go.work`.

> Named `devplus-workspace` (not `DevPlus`) to avoid a macOS case-insensitive collision with the unrelated `~/Documents/DevPlus` Obsidian vault.

**17 platform repos**, grouped:

| Group | Repo | What it is |
|---|---|---|
| **Service** | `devplus-backend` | Main API service (REST + WebSocket) |
| **Service** | `devplus-executor` | OpenTofu plan/apply + Infracost runner (dispatcher + job pods) |
| **Service** | `devplus-oidc-service` | OIDC issuer / JWT token-exchange |
| **Service** | `devplus-opentofu-server` | OpenTofu module registry + HTTP state backend |
| **Service** | `devplus-governance-service` | Cedar policy evaluation |
| **Service** | `devplus-agent-service` | Python AI agent service (FastAPI + LangGraph) |
| **Service** | `devplus-frontend` | React 19 SPA (Vite) served by nginx |
| **Shared** | `devplus-shared` | Shared Go library module (`github.com/DevPlusInc/devplus-shared`) |
| **Platform** | `devplus-platform` | Tilt + Helm chart + tenant terraform + CI (orchestration hub) |
| **Infra** | `terraform-devplus-tenant` | Deploys a single tenant to Kubernetes |
| **Infra** | `terraform-helm-devplus-services` | Reusable module that Helm-deploys all services |
| **Infra** | `terraform-cloudflare-ingress` | Per-namespace Cloudflare Zero Trust tunnel ingress |
| **Infra** | `devplus-cicd-infra` | Azure AD apps + GitHub OIDC federated creds for CI/CD |
| **Infra** | `devplus-github-settings` | SOC2 GitHub org controls via Terraform |
| **Infra** | `devplus-hub-infra` | Centralized hub VNet, runners, NAT, private DNS, Key Vaults |
| **Infra** | `devplus-base-infra` | Terraform/OpenTofu remote-state backend (Storage + tfstate) |
| **Infra** | `devplus-core-infra` | Per-env core infra: VNets, private AKS, MongoDB Atlas, AI Foundry |

## 3. One-Time Setup

1. **Clone the repos** as siblings under the umbrella parent `/Users/helderfeixas/Documents/devplus-workspace`.
2. **SSO for `devplus-shared`** — it is **SAML-SSO protected**. Authorize the GitHub CLI token for the `DevPlusInc` org *before* cloning it (the `go.work` replace target every Go service needs).
3. **Azure access for Key Vault**:
   - `az login` (subscription **`devplus-dev`**).
   - **Cloudflare WARP must be ON**: Key Vault `kv-devplus-dev` has public network access disabled (default deny) and is reachable only through its private endpoint in the hub VNet, which WARP reaches over the hub cloudflared tunnel with privatelink DNS. The same holds for the state storage account (`devplus-base-infra`).
4. **`finish-setup.sh`** (in the umbrella) — clones `devplus-shared`, regenerates `go.work`, and builds every Go service.
   - This produces the umbrella `go.work` linking `devplus-shared` + the 5 Go services for local build/test. `go.work` is gitignored inside each repo and is **not** used in CI/Docker.
5. **k3d cluster for local mode** (one-time):
   ```
   k3d cluster create dev --registry-create k3d-registry:0.0.0.0:5111
   ```

### Toolchain (expected on the dev machine)

`tilt`, `k3d`, `kubectl`, `helm`, `az`, `tofu` (OpenTofu), `docker`, `go 1.26.1`, `node`, `bun` (frontend), `uv` (agent-service), `python3` (`cf-ingress.sh`), `gh`, `atlas` (MongoDB Atlas CLI), `mongosh`.

> There is **no standalone `terraform` binary** — OpenTofu is used. Infracost runs inside the executor image.

## 4. Running It

Always run **from the umbrella parent directory**:

```
tilt up -f devplus-platform/Tiltfile -- --tenant=<name> --mode=<local|atlas|aks> [--tag=<ver>] [--llm=<foundry-deployment>] [--reasoning=<high|medium|low|off>]
```

`--llm` defaults to `DeepSeek-V4.1-Flash`, `--reasoning` to `high` (both agent-service only; switch live with `tilt args`). `OTEL_ENABLED=true` in the `tilt up` environment turns on trace export.

Tilt UI: **http://localhost:10350**

### Modes

| Mode | Cluster | MongoDB | Registry / Images | `--tag` | Notes |
|---|---|---|---|---|---|
| `local` | k3d `dev` | In-cluster MongoDB (`admin`/`admin123`) | local registry `localhost:5111` | optional | Default fast loop |
| `atlas` | k3d | MongoDB Atlas (creds from Key Vault) | local registry | optional | Real Atlas data plane |
| `aks` | `aks-devplus-dev-eus2` | MongoDB Atlas via OIDC workload identity | pushed to GHCR | **required** (e.g. `0.1.0-dev1`) | Cloud target |

### Prerequisites (all modes)

- `az login` (subscription `devplus-dev`).
- **Cloudflare WARP ON** to reach the network-locked Key Vault `kv-devplus-dev`.
- Tilt pulls `workos-*`, `infracost-api-key` and the optional `hubspot-access-token` from that Key Vault in **all** modes. local/atlas only: `github-app-key`, `ghcr-puller-*`, and a host-minted `aoai-token` (`az account get-access-token`); in aks the GitHub App and GHCR pull secrets are core-infra Reflector mirrors and AI calls use the workload identity. atlas also reads `mongodb-atlas-username`/`-pw`; aks reads `cloudflare-terraform-token` (`cf-ingress.sh`).

### AKS-mode extra steps

```
gh auth refresh -h github.com -s write:packages,read:packages
gh auth token | docker login ghcr.io -u <login> --password-stdin
atlas auth login
az aks get-credentials -g devplus-dev-eus2 -n aks-devplus-dev-eus2   # kube context aks-devplus-dev-eus2
```

### AKS tenant ingress (Cloudflare)

In `aks` mode, Tilt provisions a per-tenant **Cloudflare tunnel** (parity with `terraform-cloudflare-ingress`): the `cloudflare-ingress` resource (`scripts/cf-ingress.sh`) creates the tunnel + a proxied DNS record and runs `cloudflared` in the namespace, exposing the tenant at **`https://<tenant>.dev.devplus.io`** (`/api/v1/*`→backend, `/*`→frontend). `tenantHost` is set to that host (not `localhost`). The `workos-org-mapping` resource (`scripts/seed-workos-org.sh`) seeds the auth-service `org→instanceUrl` mapping so the WorkOS login round-trip returns to the tenant.

- Open **`https://<tenant>.dev.devplus.io/`** — turn **WARP off** for the browser (the host is on Cloudflare's public edge; WARP routes `*.devplus.io` to a private path). WARP must be **on** during `tilt up` to read the Cloudflare token from Key Vault.
- **Teardown:** trigger the manual **`cloudflare-ingress-down`** button in the Tilt UI (or `devplus-platform/scripts/cf-ingress.sh down <tenant> dev <ns>`) **before** `tilt down` — Tilt can't remove the Cloudflare-side tunnel/DNS on `down`.

### Ports

| Service | Port | Notes |
|---|---|---|
| frontend | 3000 | local; exposed as Service `:80` |
| backend | 8080 | |
| oidc-service | 8081 | |
| opentofu-server | 8082 | |
| agent-service | 8083 | |
| governance | 8084 | |
| devplus-mcp | 8085 | chart-optional (`devplusMcp.enabled`); not deployed by Tilt |
| mongodb | 27017 | local mode only |

> `executor-job-dispatcher` is a Deployment with **no Service / no port**.

## 5. Per-Service Reference

| Service | Language | Port | Role | Run standalone (from umbrella root) | Depends on |
|---|---|---|---|---|---|
| **backend** | Go 1.26.1 | 8080 | Main REST + WebSocket API: projects, deployments, catalog/modules, connectors, users/groups/SSO, RAG/search, AI-assistant proxy, support desk (HubSpot tickets) | `go build -o backend ./devplus-backend && ./backend` (loads local `.env`; needs MongoDB) | MongoDB (incl. the envelope-encrypted `connector_secrets` store; `SECRET_ENCRYPTION_KEY`), oidc-service, governance, agent-service, devplus-shared, WorkOS, HubSpot (optional), Azure OpenAI/Ollama/OpenAI, OTLP |
| **executor** | Go 1.26.1 | — (no HTTP) | Runs OpenTofu plan/apply + Infracost; dispatcher watches Mongo queues and spawns K8s job pods, or runs one-shot as a job pod | `go build -o executor ./devplus-executor`; `EXECUTOR_MODE=dispatcher ./executor` (needs Mongo) or `EXECUTOR_MODE=job DEPLOYMENT_ID=<hex> ./executor`; runtime needs `tofu`/`helm`/`kubectl`/`infracost` on PATH | MongoDB, K8s API, opentofu-server, backend, oidc-service, Infracost, GitHub App, AWS STS / Azure KV, devplus-shared |
| **oidc-service** | Go 1.26.1 | 8081 | OIDC issuer: RS256 JWTs (user/service/project/deployment), discovery + JWKS, token validation, K8s SA TokenReview exchange | `go build -o oidc-service ./devplus-oidc-service && ./oidc-service` (env-only; `/token/service` needs in-cluster TokenReview, all other endpoints work standalone) | devplus-shared (otel), K8s TokenReview API; consumed by backend/executor and by cloud workload-identity federation (AWS/Azure/GCP trust its JWKS). **No database**. Wires everything in `main.go` (no `internal/app`) |
| **opentofu-server** | Go 1.26.1 | 8082 | OpenTofu module registry (Terraform Registry Protocol) + HTTP state backend; modules in GridFS, state in Mongo | `go build -o opentofu-server ./devplus-opentofu-server`; `PORT=8082 MONGODB_URI=... ./opentofu-server` (needs MongoDB; `GET /health`) | MongoDB, devplus-shared, OTLP (optional) |
| **governance-service** | Go 1.26.1 | 8084 | Cedar policy evaluation: allow/deny on deployment/module actions, returns blocking policy names; loads `governance_policies` (`enabled:true`) | `go build -o governance-service ./devplus-governance-service && ./governance-service` (Mongo optional — zero policies ⇒ deny-all) | MongoDB (optional), devplus-shared, OTLP (optional); backend calls `POST /reload` |
| **agent-service** | Python (≥3.11; Docker py3.12) / uv | 8083 | FastAPI + LangGraph/LangChain agents (Azure Foundry via Entra ID: DeepSeek-V4.1-Flash in Tilt, `--llm=` to override; gpt-5.4 in deployed envs) for chat over catalog/cost/health/deploy/update/delete, architect + adoption agents (discovery scans run in the executor, §14) | `uv sync && uv run python -m src.agent_service.api.main` (uvicorn `:8083`, reload); tests `uv run pytest` | backend, governance, Azure OpenAI (no cloud SDKs) |
| **frontend** | TypeScript (React 19 SPA, Vite 7) / Bun | 3000 | Web UI; pure UI layer, no DB — static Vite bundle served by nginx, which reverse-proxies `/api/v1/*`, `/callback`, and `/.well-known/` to backend/oidc-service same-origin; auth is a sealed httpOnly `devplus_session` cookie (no token ever in JS) | `bun install && bun run dev` → `<tenant>.localhost:3000` (Vite dev proxy mirrors the nginx routes); build `bun run build` (nginx serves `dist/`); tests `bun run test` (vitest) | backend + oidc-service (via nginx/Vite proxy), WorkOS (via backend) |

> Service repos have **no Makefile** and **no committed `.env.example`** (`.env` is gitignored; backend/agent-service load it via godotenv/settings). Go services also have **no in-repo Tiltfile** — Tilt lives in `devplus-platform`.

## 6. The Go Workspace Model

| Concept | Detail |
|---|---|
| `go.work` | Lives in the umbrella parent; links `devplus-shared` + the 5 Go services for local build/test. **Gitignored inside each repo; never committed; not used in CI/Docker.** |
| `devplus-shared` | Private Go module `github.com/DevPlusInc/devplus-shared`, SemVer-tagged (latest `v1.16.0`; consumers pin exact tags: backend/executor `v1.16.0`, oidc/opentofu-server/governance `v1.14.0`). Provides models, MongoDB helper, `crypto` (envelope sealer), OTel bootstrap, `gcpauth`/`gcpsm`, HTTP/string utils. Not runnable. SAML-SSO protected. |
| `Dockerfile` (prod/CI) | Fetches the private `devplus-shared` via a GitHub App token mounted as BuildKit secret `gh_token` (e.g. `docker build --secret id=gh_token,src=token.txt ...`). `GOPRIVATE=github.com/DevPlusInc`. |
| `Dockerfile.tilt` (local) | Built with the **umbrella root as build context** (`..`), copies the sibling `../devplus-shared` in, then `go mod edit -replace github.com/DevPlusInc/devplus-shared=/devplus-shared` + `go mod tidy` — so local edits to shared are picked up with no tag bump or token. |
| Build split | Go services that need shared build via `Dockerfile.tilt` from the umbrella root. **frontend** and **agent-service** build from their own dir with their production `Dockerfile` (they have no `Dockerfile.tilt`). `devplus-mcp` is not in Tilt's image list. |
| Hot reload | **No Tilt `live_update`/sync is configured for any service.** Each source change triggers a full image rebuild + pod redeploy. Pure Go work can use the umbrella `go.work` with plain `go build` / `go test`. |
| `finish-setup.sh` | Clones `devplus-shared`, regenerates `go.work`, builds every Go service. |

## 7. MongoDB Authentication

### AKS — Azure Workload Identity / `MONGODB-OIDC` (passwordless)

| Step | Mechanism |
|---|---|
| ServiceAccount | Each tenant's pods run under a K8s ServiceAccount annotated with an Azure user-assigned managed identity client-id and labeled `azure.workload.identity/use=true`. |
| Federated token | The Azure workload-identity webhook projects `AZURE_FEDERATED_TOKEN_FILE` — issuer = AKS cluster OIDC issuer, audience `api://AzureADTokenExchange`, subject `system:serviceaccount:<ns>:<sa>`. |
| Managed identity | Has federated credentials registered for those subjects against the AKS OIDC issuer. |
| Atlas IdP | Per-cluster **WORKLOAD OIDC Identity Provider** (issuer = AKS OIDC issuer, audience `api://AzureADTokenExchange`, `authorizationType USER`, `userClaim sub`) connected to the Atlas org. |
| Atlas DB users | Named `<idpId>/system:serviceaccount:<ns>:<sa>`. |
| Connection string | `mongodb+srv://<cluster>.mongodb.net/?authSource=$external&authMechanism=MONGODB-OIDC&authMechanismProperties=ENVIRONMENT:k8s` — **no passwords**. |

Where the wiring lives:

| Concern | Repo |
|---|---|
| Atlas IdPs, clusters | `devplus-core-infra` |
| Per-tenant identity, federated credentials, DB users | `terraform-devplus-tenant` |
| Org-level federation config | federation layer |

### local / atlas

In **local** mode MongoDB is an in-cluster **StatefulSet** with static `admin`/`admin123` credentials. In **atlas** mode services use MongoDB Atlas with Atlas creds sourced from Azure Key Vault.

## 8. Helm Chart, Image Tags & GHCR

The umbrella Helm chart at **`devplus-platform/k8s/helm/devplus`** deploys all services into a per-tenant namespace (source `Chart.yaml` stays `0.1.0`; the Release workflow stamps the real version). GHCR images are `ghcr.io/devplusinc/<key>`. **One SemVer pins everything**: a published chart version == appVersion == every image tag (`image.tag` defaults to appVersion), and tenants deploy a `chart_version` from `oci://ghcr.io/devplusinc/charts/devplus`. `image-tags.json` was removed on 2026-07-09. Release promotes **8** images by digest (`crane copy`).

| Image key (GHCR) | Kubernetes resource |
|---|---|
| `backend` | Service `backend` |
| `executor` | Deployment `executor-job-dispatcher` (**no Service**) |
| `governance-service` | Service `governance` |
| `opentofu-server` | Service `opentofu-server` |
| `oidc-service` | Service `oidc-service` |
| `agent-service` | Service `agent-service` |
| `frontend` | Service `frontend` |
| `devplus-mcp` | Service `devplus-mcp` (:8085, only when `devplusMcp.enabled`) |

In-cluster helpers shipped by the platform (core-infra on AKS; Tilt installs them into k3d for local/atlas):

| Helper | Role |
|---|---|
| **Stakater Reloader** | Rolls deployments on secret/configmap change. |
| **emberstack Reflector** | Mirrors source secrets (GitHub App, GHCR pull, `github-install-state-shared`, `hubspot-access-token-shared`) into tenant namespaces. |

## 9. Tenant Model, WorkOS Auth & Azure OpenAI

### Tenant model

**tenant = Kubernetes namespace = MongoDB database name.** Locally the browser host is `<tenant>.localhost:3000` (Chrome auto-resolves `*.localhost`). Each tenant has its own **WorkOS `organizationId`**.

### Auth (WorkOS)

WorkOS **AuthKit** is the user auth provider (handled by the backend). The redirect URI is centralized at `auth.dev.devplus.io/callback`. WorkOS keys come from Key Vault. Tenant-host gating is enforced by the frontend's nginx via the `TENANT_HOST` env var (`$tenant_ok` map; unmatched hosts get 404, `/healthz` and `/.well-known/` exempt).

### Azure OpenAI

Backend (RAG) and agent-service call Azure OpenAI with **Entra ID tokens (no API keys)** via the **`aoai-token`** secret — host-minted with ~1h expiry.

> If AI calls start returning **401**, use the **`aoai-token` refresh button in the Tilt UI**.

The app workload identity is granted **`Cognitive Services OpenAI User`** on the `ais-devplus-<env>` account. That role covers OpenAI-format models only: non-OpenAI deployments (DeepSeek-V4.1-Flash, FW-GLM-5.3-Flash and the rest of the `ai-foundry-models.tf` catalogue) need **`Azure AI Developer`** (`MaaS/*`), which core-infra grants only to the admins group and the Foundry project; through the OpenAI surface they return a bare 404.

## 10. Infra / Terraform Repos

| Repo | Purpose |
|---|---|
| `terraform-devplus-tenant` | Deploys a single tenant: WorkOS org, namespace, secrets, Helm release (via `terraform-helm-devplus-services`), optional Atlas OIDC users and Azure workload identity, and (always) the Cloudflare tunnel ingress. Host: `<tenant>.<env>.devplus.io`, bare `<tenant>.devplus.io` in prod. |
| `terraform-helm-devplus-services` | Reusable module deploying all services via Helm: namespace, core secrets (mongodb/jwt/internal-api, `secret-encryption-key`, `workos-*`, `github-install-state`, `mongodb-atlas-uri`; `ghcr-pull-secret` only when `ghcr_token` is set, tenants use the Reflector mirror), optional PVCs, Helm release of the published OCI chart pinned by `chart_version`. |
| `terraform-cloudflare-ingress` | Per-namespace Cloudflare Zero Trust tunnel ingress: Tunnel, config (anchored `.well-known` rules to oidc-service, `/mcp` + `oauth-protected-resource` to devplus-mcp, `/api/v1/*` to backend, rest to frontend), DNS CNAME, `cloudflared` Deployment + token Secret. |
| `devplus-cicd-infra` | Entra app registrations + SPs for GitHub Actions (dev/stage/prod): environment-scoped and flexible (`repo:<org>/<repo>:*`, dev app) federated credentials, **Owner** on each env subscription plus hub/monitoring/state grants, and the prod DevPlus connector principal (ABAC-limited RBAC Admin). No CI; apply from a workstation. |
| `devplus-github-settings` | SOC2 GitHub org controls: least-privilege base perms, default-branch ruleset, Actions allowlist (explicit patterns, no verified-creator wildcard), secret scanning / Dependabot defaults, the Founders team + `all_repo_admin` role. No CI (local apply by an org owner). **The live ruleset has drifted from code** (approvals, signatures, bypass list, Actions allowlist): reconcile before any apply, or it reverts live settings (README "Live drift"). |
| `devplus-hub-infra` | Centralized hub VNet (cloudflared tunnel, shared runners, NAT gateway, spoke peering), private DNS zones (incl. AMPLS), per-env private Key Vaults `kv-devplus-<env>` (each in its own `devplus-keyvault-<env>` RG; dev/stage in their own subscriptions), AMPLS, hub Log Analytics (365-day retention)/Sentinel/Network Watchers. Deploys on push to main and `workflow_dispatch` (no PR plan); prod environment needs a Founders reviewer. |
| `devplus-base-infra` | Terraform/OpenTofu remote-state backend: Azure Storage Account `devplusstateprod` + tfstate blob container (Entra ID auth, shared keys disabled). **Private-endpoint only since 2026-09-20** (public access off, bypass None, PE in the hub): every local tofu run needs a network path to it. |
| `devplus-core-infra` | Per-env core infra: regional VNets/subnets/NAT/NSGs, private AKS (Cilium + Karpenter; MMK disks, `prevent_destroy`), MongoDB Atlas (M10 in every env), AI Foundry + models + catalogue, Defender (Sentinel is hub-owned), alerts, in-cluster platform services (github-callback, auth-service, OTel collector, external-secrets, Reflector, Reloader, Kyverno), prod-only Cloudflare WAF/R2. |

## 11. Conventions & Key Files

| File / Path | Role |
|---|---|
| `go.work` (umbrella) | Local Go multi-module workspace. Untracked; never committed. |
| `finish-setup.sh` (umbrella) | One-shot setup: clone shared, regen `go.work`, build Go services. |
| `devplus-platform/Tiltfile` | Local-dev orchestration (Starlark): `--tenant`/`--mode`/`--tag` parsing, image builds, Helm values, AKS/Atlas/federated-identity setup. |
| `devplus-platform/k8s/helm/devplus/` | Umbrella Helm chart deploying all services per tenant. |
| `devplus-platform/CHANGELOG.md` | Release record: one entry per published chart version. |
| `devplus-platform/terraform/tenants/devplus/environments/` | Per-tenant, per-env OpenTofu configs (`dev|stage|prod`, per-dev `helder|aaron|[[Saamer|saamer]]`, canary `test-<env>`). |
| `devplus-platform/scripts/` | `bootstrap-db.sh`, `create-admin.sh`, `clear-registry.sh` (these three still hardcode namespace `devplus`), `cf-ingress.sh` (aks Cloudflare tunnel up/down), `seed-workos-org.sh` (auth.workos mapping), `publish-chart.sh`, `check-shared-vectors.sh`. |
| `devplus-platform/{TILT.md, README.md, DATABASE_OPERATIONS.md}`, `devplus-platform/docs/` | Onboarding / runbook docs (`CICD.md`, `RELEASE.md`, cloud setup guides, `SUPPORT_DESK.md`, `GITHUB_REPO_ANALYSIS.md`). |
| `Dockerfile` vs `Dockerfile.tilt` (Go service repos) | Prod (private-module via `gh_token`) vs local-Tilt (umbrella context + shared replace). |
| `internal/app/app.go` (Go services) | Wiring entrypoint called from `main.go` (oidc-service wires in `main.go` directly). |

Conventions:
- **OpenTofu, not Terraform** — there is no standalone `terraform` binary; use `tofu`.
- All service images build **`linux/arm64`**.
- Service-to-service auth uses oidc-service-minted JWTs and an `INTERNAL_API_TOKEN` / `X-Internal-Token`.
- Secrets are sourced from Key Vault (Tilt) or via Reflector/external-secrets (cluster); inline secret values and volatile GUIDs are **not** committed — concrete values live in the `Tiltfile` and terraform.

## 12. CI / Deploy

### Per-service CI (`.github/workflows/build.yml` in each service repo)

| Stage | Detail |
|---|---|
| Lint / vuln | Go: `go vet`, `golangci-lint v2.12.2`, `govulncheck`. Python: `pip-audit` + `ruff`. Frontend: `bun audit` + `eslint`. |
| Test + coverage | Hard **80% coverage gate** scoped to each repo's designated logic-layer files. |
| Build / push | `linux/arm64` image to `ghcr.io/devplusinc/<key>` (`:<full commit sha>` + `:latest`, only on a main push or manual dispatch); GitHub-native **SBOM + SLSA** attestations. Backend adds spec gates and a schemathesis conformance job. |
| Platform dispatch | `repository_dispatch` (`service-image`) to `DevPlusInc/devplus-platform`. It **deploys nothing**: it runs `platform-test-matrix.yml` (hermetic suite) with that service at its pushed branch. devplus-mcp does not dispatch. |

### Platform CI/deploy (`devplus-platform/.github/workflows/`)

| Workflow | Role |
|---|---|
| `ci.yml` | Helm lint/template + Terraform (OpenTofu) `fmt`/`validate`. |
| `platform-test-matrix.yml` | Hermetic black-box suite (`tests/`): PRs, pushes touching `tests/`, nightly, and `service-image` dispatches. |
| `release.yml` | **Actions → Release**: resolves/takes a version, `crane copy` of all 8 images by digest, chart push to `oci://ghcr.io/devplusinc/charts/devplus`, optional deploy. |
| `deploy-tenants.yml` | Manual deploy of a published `chart_version`: inputs `environment`, `tfvars_name`, `tenants`, `plan_only`, `quality_gate`. |
| `_deploy-environment.yml` | Reusable OpenTofu plan/apply matrix against the tenant (private `terraform-devplus-tenant` module); state key `tenants/<tfvars_name>.tfstate`; `quality-gate` job runs `tests/` against the `test` canary. |
| `dependency-review.yml` | Fails PRs on moderate+ severity dependencies. |

Deploys go **canary first**: `tfvars_name=test-<env>` deploys the `test` tenant (`test.<env>.devplus.io`, prod `test.devplus.io`), then the quality gate (dev smoke + contract, stage + regression + capped k6, prod smoke) must be green before promoting. Runbooks: `devplus-platform/docs/{CICD,RELEASE}.md`.

Reproduce platform CI validation manually:

```
helm lint ./k8s/helm/devplus
helm template devplus ./k8s/helm/devplus --namespace devplus
tofu fmt -check -recursive
tofu init -backend=false && tofu validate   # in terraform/tenants/devplus
```

## 13. Hard-Won Conventions & Gotchas

Non-obvious, cross-cutting facts that have repeatedly caused bugs. Check these first when debugging 401s, OIDC, connectors, the AI feature, or GitHub App install.

### Frontend → backend auth (SPA + sealed cookie; the old `getBackendHeaders` rule is retired)
The frontend is now a pure static SPA (Vite + React 19, `src/routes.tsx` + the `app/**/page.tsx` routes) with **no `app/api/**` route handlers** — the Go backend owns all session handling. The browser sends a sealed httpOnly `devplus_session` cookie same-origin; every mutation MUST also send the `X-DevPlus-Csrf: 1` header (CSRF protection — a cross-site form POST cannot set a custom header). `/api/v1`, `/callback`, and `/.well-known/` are proxied by nginx (in-cluster) or the Vite dev server (local). The old per-route `getBackendHeaders()` 401 failure class no longer exists — keep the two proxy lists in sync if a new backend prefix is added. Full detail: `devplus-frontend/docs/SPA_MIGRATION.md`.

### Tenant-host convention and the OIDC issuer (drift → OIDC & GitHub bugs)
Both deploy paths now use the **same** form (Terraform changed in terraform-devplus-tenant 9ad7ff7, 2026-06-25), but older notes, code comments and some core-infra variable descriptions still show the retired `<tenant>-<env>.devplus.io` form:
- **Tilt aks:** `<tenant>.<tier>.devplus.io` (e.g. `helder.dev.devplus.io`), `Tiltfile` `tenant_public_host`.
- **Terraform:** `<tenant>.<env>.devplus.io`, bare `<tenant>.devplus.io` in prod (e.g. `app.devplus.io`), `host_subdomain` in terraform-devplus-tenant `main.tf`.
- **Tilt local/atlas:** `publicUrl` is `https://<tenant>.devplus.io`, which nothing routes to k3d, so cloud OIDC connectors only work in aks mode or on Terraform-deployed tenants.

The OIDC issuer's single source of truth is `OIDC_ISSUER_URL` = Helm `publicUrl` (drives the token `iss` and `/.well-known` discovery). It MUST equal the public host AWS/Cloudflare actually serve or AWS `AssumeRoleWithWebIdentity` fails. For aks, `/.well-known/openid-configuration` + `/.well-known/jwks.json` must be tunnel-routed to `oidc-service:8081` (`cf-ingress.sh` / `terraform-cloudflare-ingress`); otherwise they fall through to the frontend and 404. The `github-callback` hub's `tenant_hostname_pattern` likewise must accept the Tilt `<name>.<env>` form.

### Connector update (PUT) auth-method flags (FIXED 2026-07-18)
Historically the AWS/Azure `Update` handlers reset boolean auth flags from the body, so a `PUT` that omitted them silently disabled the auth method. Since devplus-shared adbaf8e the AWS, Azure and GCP update requests take `useOidc` / `useManagedIdentity` as tri-state `*bool`: omitted keeps the stored value, and a merged state with no auth method returns 400 (backend `handlers/providers/{aws,azure,gcp}.go`). Connector `status` is persisted by the `Test` endpoints (`connected`/`error`); PUT still accepts a non-empty `status`, so don't send one casually.

### Azure OpenAI (Foundry) is AAD-only
`ais-devplus-<env>` has `local_auth_enabled = false` (no API keys). Callers need an Entra token whose principal holds **`Cognitive Services OpenAI User`** on the account. In aks, backend/agent-service authenticate as the workload/managed identity (do NOT inject a host-minted `aoai-token` there — that's gated to local/atlas in the Tiltfile). 401 `PermissionDenied` here = missing role assignment, not bad code.

### GitHub App install
The App's **Setup URL** (Post-installation), not the OAuth Callback URL, drives the post-install redirect (DevPlus uses the App-installation flow with `installation_id`+`state`, not OAuth `code`). It points at the centralized hub `https://connect.<tier>.devplus.io/github/callback` (moved off `app.<tier>` 2026-07-11: that host is reserved for the `app` **tenant**; the hub prefix is `github_callback_subdomain_prefix` in core-infra and must never be a tenant name), which HMAC-verifies the signed `state` (shared secret), validates the tenant host against `tenant_hostname_pattern`, then 302s to `https://<tenant-host>/api/v1/connectors/providers/github/callback` (prod hub host is `connect.devplus.io`, no tier segment). In Tilt the tenant must sign with the hub's real secret (read from the `github-callback` namespace), not a placeholder. A `%20`/trailing space in the Setup URL → 404; `{"error":"invalid state"}` → wrong/expired HMAC secret; "tenant not allowed" → host fails the pattern.

### Support desk (HubSpot) — agents reply by EMAIL, notes are internal
The in-product support desk (`/support`, backend `SupportService` + `HubSpotClient`, runbook `devplus-platform/docs/SUPPORT_DESK.md`) files user requests as tickets in DevPlus Inc.'s HubSpot portal. Two things bite: (1) **a HubSpot note is never shown to the user** — only notes DevPlus wrote itself (they start with `DevPlus portal comment · …`) and **emails logged on the ticket** appear in the user's timeline, so agents must reply from the ticket by email, not by note; (2) the private-app token is **platform-wide** (one portal for all tenants) and reaches tenants as the Reflector-mirrored `hubspot-access-token-shared` (core-infra `hubspot.tf`, `hubspot_enabled` per env; Tilt in every mode creates its own per-tenant `hubspot-secret` from Key Vault, so dev tenants never wait on core-infra) — the tenant boundary is the `devplus_tenant` filter + requester check in `SupportService`, so never expose raw HubSpot reads without it, and never create another Secret with the mirror's name. No token = `/support/status` `configured:false`, sidebar entry hidden, other support routes 503 (by design, not a bug).

### GCP connectors (added 2026-09-05)
GCP is the third cloud, at parity with AWS/Azure (`type = "gcp"`). Facts that differ from the other two and bite: (1) **two audience forms**: the JWT `aud` is `https://iam.googleapis.com/projects/<num>/locations/global/workloadIdentityPools/devplus/providers/devplus-<slug>` (a WIF provider's default allowed audience) while the STS exchange `audience` parameter is the same resource **schemeless** (`//iam.googleapis.com/...`); `Connector.GCPOIDCAudience()` vs `GCPSTSAudience()`. (2) **One connection, one provider, one SA**: the pool `devplus` is shared per project, the OIDC provider `devplus-<slug>` (attribute condition `assertion.sub == 'devplus:<slug>'`) and service account `devplus-<slug>@<project>.iam.gserviceaccount.com` are per connector; the SA is REQUIRED for WIF (federated tokens are refused by some APIs and have no quota project). (3) GCP connectors **require a slug ≤ 22 chars** (`ValidateGCPSlug`; SA ids max 30) and the **project NUMBER** (not derivable from the id). (4) The **service-account key is sealed** in `connector_secrets` (`GCPServiceAccountKeySecretKey`), never on the connector doc; the doc carries only `gcpHasServiceAccountKey`; the executor unseals it via internal `GET /internal/connectors/{id}/gcp-credentials`. (5) Credentials reach tofu as env only (`GOOGLE_PROJECT` + `GOOGLE_OAUTH_ACCESS_TOKEN` or `GOOGLE_CREDENTIALS`; modules self-configure `provider "google" {}`); no gcloud in the executor image. (6) Discovery = Cloud Asset Inventory (`cloudasset.googleapis.com` + `roles/cloudasset.viewer`), types `gcp:<assetType>`, zones folded into regions; adoption import ids are the `projects/...` path (GKE `/zones/`→`/locations/` tried too). (7) All Google HTTP is stdlib: auth + Secret Manager in `devplus-shared/gcpauth` + `gcpsm`, the Cloud Asset scanner in devplus-executor `internal/gcpscan` (no Google SDK in any `go.mod`). Runbook: `devplus-platform/docs/GCP_SETUP.md`.

### GitHub repository discovery and analysis (added 2026-09-21)
GitHub is a Discovery provider whose unit is a repository, not a cloud resource. Deep reference: [`docs/GITHUB_REPO_ANALYSIS.md`](docs/GITHUB_REPO_ANALYSIS.md); runbook: `devplus-platform/docs/GITHUB_REPO_ANALYSIS.md`. Facts that bite: (1) **GitHub connectors have NO owning project** (the App install flow never sets `project_id`), so the admin pages read through the connector (`/api/v1/github/connectors/{id}/...`); the same data is served under `/projects/{pid}/github/...` for members and agents. Never resolve "the connector's project" for a GitHub connector. (2) **Two passes**: Pass A (inventory, `JOB_KIND=discovery`, no clone: list, one bulk GraphQL query per 25 repos for head SHA / `isEmpty` / exact counts, then REST branch + contributor lists for the most recently pushed) and Pass B (`JOB_KIND=repo_analysis`, one pod per repo, shallow clone, deterministic engine, no LLM anywhere). Each pass writes its own `repo_analyses` document (`source: api` for Pass A, `fs` for Pass B) and the ledger's `analysis.analysisId` points ONLY at the `fs` one, so the `git`/`authors` facets (committers) must be located through the `api` document, the way the ecosystem graph reader (`GET .../ecosystem-graph`, the Insights tab) does. (3) **GitHub wire shapes that were guessed wrong once**: a branch's SHA is `commit.sha` (not top-level); `?protected=` on the branch list is a FILTER; repository `permissions` are booleans; an exhausted primary rate limit is a `403` with `X-RateLimit-Remaining: 0` and NO `Retry-After` (not a permission error); "contributor list too large" is also a 403; REST `/releases` bodies reach 25 MB (use GraphQL); `pushed_at` is not the last commit; a truly empty repo still reports `default_branch: "main"` and `size: 0` (only GraphQL `isEmpty` is authoritative); repo names may start with a dot (`.github`). Test fakes must serve captured raw JSON (`internal/githubscan/testdata/`), never the code's own structs. (4) **The analysis pod**: uid 65532 + read-only rootfs, so the executor binary lives in `/usr/local/bin` (not `/root`) and `TMPDIR=/work/.tmp` (ctags spools to a temp file); under `MONGODB-OIDC` it must run as `<ns>-app` with the workload-identity label because Atlas has no user for the dedicated `<ns>-repo-analyzer` account (the signal is the URI's mechanism, NOT `EXECUTOR_JOB_WORKLOAD_IDENTITY`, which is true in every Tilt mode). (5) **Scoped token**: negotiated by permission RANK (the App holds `contents:write`; comparing for equality dropped `contents` and broke every private clone), minted only for a repository the dispatcher currently has leased, echoed grant verified. (6) **Run arithmetic**: `reposExcluded` (policy, at trigger time) is separate from the pod-outcome counters that close a run; a run's `_id` equals its `runId`; the queue is insert-only and unique on `(connectorId, repoId)`; a concurrency deferral must not spend a retry attempt. (7) go-git needs the object store in a `.git` chroot, separate from the worktree filesystem, or any repo with a top-level `config/` dir fails to clone. (8) Secret findings are their own Cedar action (`ListRepoSecretFindings`, project write tier). (9) `enabled: false` is honored by BOTH the dispatcher and the backend via `REPO_ANALYSIS_ENABLED`; `osvApiBase` empty means the vulnerabilities facet is OFF (reported not-configured), not "use the default".

### Multi-env deploys & the image-attestation gate (updated 2026-07-11)
`devplus-core-infra/deploy.yml` applies dev→stage→prod on merge to `main`: **all three now apply green** (the long-standing prod failure is fixed: hub images pinned to attested tags in stage/prod tfvars; the read-only matrix plan runs `-lock=false` on every event so it can't starve the deploy's state lock; deploy also runs on `workflow_dispatch`). The Kyverno image gate (`verify-ghcr-images`) is **Audit in every env for now** (tfvars set `Audit`; the variable default is `Enforce`, and some code comments still say stage/prod Enforce): the `Enforce` policy rejected provably-valid GH-native attestations (a `verifyImages` config issue, not an image problem); fix that before re-enabling Enforce. The pinned GitHub TUF root in `modules/image-verification/main.tf` is commented "expires 2026-08-01": check it still verifies or re-pin. Tenants deploy via `devplus-platform` Actions → Release (publishes chart + digest-preserving `crane` image promote; never `imagetools create`, which wraps a new unattested index) → Deploy Tenants (`environment` = dev|stage|prod; `test-<env>` canary first, then the quality gate). Atlas workload-OIDC has **three layers**, all now IaC'd: IdP objects + DB users (per-workspace) + the org-level `dataAccessIdentityProviderIds` connection (**prod workspace owns this single org-wide object**: clusters only load IdPs on that list, and only on a config publish after a list change). Foundry/storage private-endpoint DNS: only the **primary region's** PE registers the shared hub zone record (secondary-region zone groups caused a last-writer-wins record pointing at an IP other spokes can't route to; hub-spoke has no spoke↔spoke transit).

## 14. Discovery (Re-Architected) & the Architect Agent Roadmap

> **Supersedes the discovery descriptions in §1 / §5 / §9**, which predate the 2026-06 re-architecture. Discovery ingestion is **no longer** an in-process agent-service LLM task.

### Discovery today (shipped)
Discovery is a **deterministic Go scan job**, not an LLM agent. Backend `TriggerDiscovery` enqueues a durable `discovery_queue` item; the **executor's** discovery dispatcher (a mode of `devplus-executor`, alongside deploy/module jobs) leases it and runs an **isolated K8s Job pod** that scans the cloud, streams results into a paged `discovered_resources` collection (one doc per resource — escapes the 16MB BSON cap), then finalizes a thin `DiscoveryResult` control doc. This removed the LLM from the data path (which had caused Foundry 429s, OOM that took chat down, and total data loss on crash).
- **AWS**: per-service read-only sweep (paginated, bounded concurrency), enriched properties + tags (Resource Groups Tagging API).
- **Azure**: Azure Resource Graph (one paginated KQL query), all auth modes: service principal, managed identity, and OIDC client-assertion (sub `devplus:<slug>` from `Connector.OIDCSubject()`, plain `devplus` only for legacy slugless connectors; via oidc-service).
- **Extensible**: `internal/scan.Provider` interface; adding a cloud = a new `internal/<provider>scan` package + a case in the job's provider factory (2026-09: `internal/gcpscan` over Cloud Asset Inventory joined `awsscan`/`azurescan`). Shared models live in `devplus-shared` (`v1.1.0`: `DiscoveryQueueItem`, `StoredDiscoveredResource`, slimmed `DiscoveryResult`).
- **GitHub** (2026-09): `internal/githubscan` inventories a GitHub App installation (repositories, branches, releases, contributors as `github:*` resources, `region` = the repository owner) and feeds a second, per-repository job kind that analyzes repository CONTENT. It is analysis-only: nothing is deployed to or adopted from a repository. See §13 "GitHub repository discovery and analysis".
- The LLM returns only at **consumption** time (below).

### Consumption layer (the "RAG-like" use of discovery; in progress)
How agents USE the inventory. **Principle: structured queries beat embeddings for exact infrastructure inputs — the LLM interprets intent, retrieval stays deterministic.** Phased:
- **Phase 0 (done)** — fixed Azure resource-type strings + longest-match in the deploy param→resource map (`agent-service domain/agents/tools/discovery_param_map.py`, extracted from `deploy_tools.py`; GCP tokens added 2026-09) so deploy-form dropdowns populate (e.g. "pick a subnet in the chosen VPC"; Azure dropdowns were always empty).
- **Phase 1** — a general, **project/connector/run-scoped** backend query endpoint over `discovered_resources` (filter by type/region/tags/properties) + an upgraded `lookup_discovery_resources` tool.
- **Phase 2** — schema-aware input resolution (provider-aware, relationship-narrowed: choose a VPC → subnet options filter to it).
- **Phase 3** — deterministic insights, computed in the **executor** at finalize (e.g. `openToInternet`).
- **Phase 3.5** — **Environment Profile**: a structured environment digest (platforms in use / network topology / data stores / managed-vs-unmanaged + insights).
- **Deferred (seam preserved, NOT cut)** — per-resource embeddings / semantic search; reopen for fuzzy *environment exploration*, not for parameter mapping.

### The Architect Agent (north star — build toward it)
A planned agent that, given an existing **app / repo / natural-language description, designs the infrastructure to deploy** ("ECS or EKS?", "do we need a load balancer?"). Discovery + RAG must be built toward this. It needs RAG over **three distinct corpora — route each kind of intelligence to the corpus that serves it**:
1. **Decision knowledge** *(NEW corpus — where embeddings actually pay off)*: ECS-vs-EKS / ALB-vs-NLB tradeoffs, cost/ops, Well-Architected, internal playbooks. Index via the backend `rag_service`; design *with* the Architect Agent.
2. **Catalog / modules** *(already embedded)*: `rag_service.IndexCatalogItem` / `IndexModule`.
3. **Customer environment** *(this discovery data)*: consumed as **structured facts + the Environment Profile + insights — NOT per-resource vectors**. "Should I use EKS?" = corpus 1 (knowledge) + a *structured fact* from corpus 3 ("they already run 2 EKS clusters → prefer EKS"), never ANN over subnet/SG docs.

This is **why** discovery is built as a general, queryable, summarizable substrate (rich multi-cloud inventory + a general query API + insights + Environment Profile + an open semantic seam), not a parameter-mapping endpoint. Full implementation spec: workflow output `discovery-consumption-spec.md`.

## 15. Retired: Hermes and Paperclip (decommissioned 2026-09-23)

The Hermes ops agent (Proxmox VM 200), Paperclip (VM 105), their Teams bot, Cloudflare tunnel `devplus-hermes-agent` and every `*.tools.devplus.io` hostname were decommissioned on 2026-09-23 in favour of Helder's own harness, **Cove** (Cove.app, which calls Azure Foundry with an Azure CLI token and the Sparks on the LAN at `http://192.168.1.75:8888/v1`). Repos `devplus-hermes-agent` and `devplus-hermes-state` are retired; verified git bundles, the Paperclip DB and the VM's uncommitted work are archived in `~/Documents/devplus-archives/2026-09-23-hermes-paperclip/`, with the full inventory in `hermes-paperclip-decommission-inventory.md` at the umbrella root.

Kept and repurposed for Cove: the GitHub App (formerly "DevPlus Hermes Agent", App 4890296) and the Entra app `516f1d04` (renamed **DevPlus Cove**; delegated Microsoft 365 sign-in only, no client secret, no app-only permissions). Do not recreate Hermes-era identities, hostnames or the `agent:*` labels without a decision to bring an agent platform back.

The DGX Sparks remain model hosts only (see `docs/` and memory for the current serving recipe); nothing on them depends on Hermes or Paperclip.
