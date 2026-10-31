---
id: d92d0c7a-a3c4-4d8a-9d57-9a4755cc9b0a
binder: binder-mvvle32a-1
---
DevOps | devplus-docs/OPERATIONS.md

Source: DevPlusInc/devplus-docs/OPERATIONS.md, branch docs/devops-refresh-2026-10, refreshed 2026-10-04

Refreshed 2026-10-04 against origin/main code (DevPlusInc/devplus-docs, OPERATIONS.md).

# Operations

> Environments, CI/CD, release/promotion, tenancy operations. The daily-driver run instructions live in `CLAUDE.md` §3–4 and `devplus-platform/TILT.md` — this doc covers the machinery around them.

## 1. Environments

| Env | Subscription | AKS | Atlas | AOAI | Key Vault | Tenant host |
|---|---|---|---|---|---|---|
| dev | `devplus-dev` | `aks-devplus-dev-eus2` | `devplus-dev.wnzhnl` | `ais-devplus-dev` | `kv-devplus-dev` | `app.dev.devplus.io` (+ per-dev tenants `helder`/`aaron`/`saamer`, canary `test.dev.devplus.io`) |
| stage | devplus-stage | `aks-devplus-stage-eus2` | `devplus-stage.ibq1ym` | `ais-devplus-stage` | `kv-devplus-stage` | `app.stage.devplus.io` (+ canary `test.stage.devplus.io`) |
| prod | devplus-prod | `aks-devplus-prod-eus2` | `devplus-prod.na05gc` | `ais-devplus-prod` | `kv-devplus-prod` | `app.devplus.io` (**control-plane tenant**: `control_plane_enabled=true`; plus tenants provisioned from inside DevPlus, e.g. `dan.devplus.io`; canary `test.devplus.io`) |

Hub (prod subscription): hub VNet + peering, cloudflared private access (replaces VPN), centralized private DNS zones, per-env Key Vaults, VNet-injected GitHub runners, Log Analytics + Sentinel. AI Foundry model deployments (source of truth `devplus-core-infra/ai-foundry.tf`): **gpt-5.4** (the deployed-env chat default), **gpt-5.4-mini**, **text-embedding-3-small**, plus the pay-per-token catalogue in `ai-foundry-models.tf` (no Marketplace models; includes **DeepSeek-V4.1-Flash**, the Tilt agent-service default). Remember the 3-layer deployment-name default and that terraform `--set` beats chart values ([[aoai-model-deployment-drift]]).

## 2. Local dev (Tilt) — mode cheat sheet

`tilt up -f devplus-platform/Tiltfile -- --tenant=<t> --mode=<local|atlas|aks> [--tag=<v>]` from the umbrella root. WARP ON for Key Vault; `az login` on `devplus-dev`.

| Mode | Mongo | Registry | Extras |
|---|---|---|---|
| local | in-cluster StatefulSet (admin/admin123) | k3d `localhost:5111` | Tilt installs Reloader/Reflector, creates github-app + ghcr secrets |
| atlas | Atlas (user/pass from KV) | k3d | same |
| aks | Atlas via **MONGODB-OIDC** workload identity | GHCR (`--tag` required, `push-*` resources) | federated creds + Atlas OIDC users + Cloudflare tunnel (`cf-ingress.sh`) + WorkOS org seeding (`seed-workos-org.sh`); AOAI via workload identity (no `aoai-token`) |

Manual Tilt buttons: `aoai-token` refresh (AI 401s in local/atlas), **`cloudflare-ingress-down` before `tilt down`**, `publish-helm-chart`. WorkOS org for tenant login is hardcoded in the Tiltfile ×2 ([[workos-tilt-tenant-org]]).

Key scripts (`devplus-platform/scripts/`): `bootstrap-db.sh` (seed/clear/indexes), `create-admin.sh`, `clear-registry.sh`, `cf-ingress.sh`, `seed-workos-org.sh`, `publish-chart.sh`.

## 3. CI (per-service)

Every service repo: lint/vuln (golangci-lint + govulncheck with `check-latest: true` / ruff + pip-audit / eslint + bun audit) → tests with **hard 80% coverage gate** (scoped per repo, see each service doc) → arm64 image to GHCR (`:<full commit sha>` + `:latest`, only on a main push or manual dispatch) with **SBOM + SLSA attestations** → `repository_dispatch` (`service-image`) to devplus-platform, which deploys nothing: it runs the hermetic Platform Test Matrix with that service at its pushed branch (devplus-mcp does not dispatch). Backend adds spec gates (stale-spec, oasdiff-breaking, consumer smoke) and a schemathesis conformance job; frontend adds the stale-API-types and color-ratchet gates; SDK has a stale-client gate.

## 4. Release & deploy (the uniform-versioning model)

Authoritative docs: `devplus-platform/docs/CICD.md` + `RELEASE.md`. Model: **one SemVer (pre-release `0.1.0-dev<N>` or an explicit one like `0.1.16-dev2`, stable `0.1.16`) == chart version == appVersion == every image tag**; a version is an artifact, not an environment; deploys are deliberate (manual workflow), CD = terraform `chart_version`.

1. **Actions → Release** (devplus-platform): resolves next version (or takes input; no-clobber on existing tags) → **`crane copy` by digest** for all 8 services (`backend executor governance-service opentofu-server oidc-service frontend agent-service devplus-mcp`) — digest-preserving so attestations survive Kyverno; never `imagetools create` → `helm package/push` to `oci://ghcr.io/devplusinc/charts/devplus`. Re-releasing an old chart: `source_tag=<X>` re-versions the *published* chart (never mixes main templates with old images — [[chart-version-oci-deploy]]).
2. **Deploy**: Release's `deploy` input or **Actions → Deploy Tenants** (`environment`, `chart_version`, `tfvars_name`, `tenants`, `plan_only`, `quality_gate`) → `_deploy-environment.yml`: per-tenant-dir matrix, GitHub Environment supplies Azure identities, secrets pulled from the env's Key Vault (note `hub-mongodbatlas-*` naming), `tofu workspace select <env>` + `apply -var-file=environments/<tfvars>.tfvars -var chart_version=<X>`.
3. **Quality gate**: deploy the `test` canary first (`tfvars_name=test-<env>`); the `quality-gate` job then runs the `tests/` black-box suite against `https://test.<env>.devplus.io` (prod `https://test.devplus.io`): dev smoke + OpenAPI contract, stage + full regression + capped k6, prod smoke only. Red blocks promotion. PAT: Actions secret `DEVPLUS_TEST_PAT_<ENV>`, else Key Vault `devplus-test-pat`. Runbook: `devplus-platform/docs/RELEASE.md`.
4. GHCR auth gotcha: GitHub App tokens don't work on the Docker registry; release uses the `ghcr-puller-pat` (classic PAT, `repo, write:packages`); helm-provider OCI pulls need explicit `registries` creds (`TF_VAR_ghcr_pull_token`).
5. **Update `devplus-platform/CHANGELOG.md` with every release cut** ([[changelog-maintenance]]).

Tenant terraform chain: `devplus-platform/terraform/tenants/devplus` (workspace = env; per-dev tfvars + `-backend-config=key=` state isolation; creates workload identity + federated creds + AOAI role) → `terraform-devplus-tenant` (WorkOS org via restapi, Atlas OIDC DB users scoped to the tenant DB, host convention) → `terraform-helm-devplus-services` (random secrets + K8s secrets — WorkOS secrets MUST be created here or helm deadlocks — + `helm_release` atomic) → `terraform-cloudflare-ingress` (tunnel, anchored ingress rules, DNS, cloudflared).

State backend: `devplusstateprod/tfstate` (Entra-auth; from devplus-base-infra). Keys: `core.tfstate` (workspaced), `hub.tfstate`, `tenants/<name>.tfstate`, `ghidentity.tfstate`, `github.tfstate`.

## 5. Infra deploys

`devplus-core-infra/deploy.yml`: checkov + tflint + matrix plan (PR comment; `-lock=false` on PR plans) → apply dev→stage→prod on merge to main — all green since 2026-07-11. Kyverno image gate is **Audit** everywhere until the Enforce/sigstore issue is fixed. Hub images (auth-service, github-callback) build in-repo with attestations; **a green core-infra apply can ship config without code — verify the running pod** ([[workos-multienv-auth]]).

## 6. Operational runbook pointers (symptom → first check)

| Symptom | First check |
|---|---|
| Everything 403s | governance `GET /policies/loaded` count (deny-all on empty/parse-fail/Mongo-down) |
| AI 401 (local/atlas) | `aoai-token` Tilt button (1h expiry) |
| AI 404 | model deployment name vs `ai-foundry.tf` (3-layer default; `--set` beats chart) |
| AI 401 PermissionDenied (aks) | `Cognitive Services OpenAI User` role, not code |
| Connector form "Unable to determine OIDC issuer" | unanchored `.well-known` ingress rules; re-apply tenant |
| AWS AssumeRole fails | `OIDC_ISSUER_URL` vs actual public host; JWKS reachable publicly |
| Azure AADSTS7000274 | oidc-service restarted with regenerated key (kid fix covers this; else wait/refetch) |
| GCP STS `invalid_grant` "attribute condition" / audience mismatch | provider pins another subject, or project number/pool/provider on the connector differ from what was created (compare the connector page's provider resource) |
| GCP 403 `generateAccessToken` | missing `roles/iam.workloadIdentityUser` binding on the connector's SA (or wrong principal); IAM propagates in ~1 min |
| GCP discovery `SERVICE_DISABLED` | enable `cloudasset.googleapis.com` + `roles/cloudasset.viewer` on the connector's SA |
| Cannot log in to tenant | WorkOS org mapping (auth hub seed) + org id in Tiltfile/tfvars |
| MONGODB-OIDC auth fails | 3 Atlas layers + org config publish ([[atlas-oidc-pe-dns-model]]) |
| Pod → Azure PE timeouts | hub DNS single-record-owner rule (primary region only) |
| Frontend 401s on proxied calls | (Next-era class — now: cookie/CSRF; check `X-DevPlus-Csrf` and refresh flow) |
| tofu state locked forever | no lock TTL: manual UNLOCK against opentofu-server, with the holder's lock `ID` from the 423 body (`DELETE` drops the state itself) |
| Deploy Tenants auth failure (chart pull, private module clone, Azure login) | `ghcr-puller-pat` validity; GitHub Environment vars (`AZURE_CLIENT_ID`, `KEYVAULT`, `AKS_*`) |
| Quality gate red | the JUnit summary on the run + `quality-gate-<env>-<run>` artifact; `DEVPLUS_PAT rejected` = the canary PAT lapsed (`devplus-platform/docs/RELEASE.md` "When it goes wrong") |
| Deploy green but the feature is missing / old code in prod | the running pod's image tag; a promotion from the wrong `-dev<N>` `source_tag` ships that artifact faithfully (RELEASE.md step 4 cross-check) |
| Provisioned tenant: "Failed to generate install URL" / "GitHub App install flow is not configured" | `github-install-state-shared` mirrored into the tenant ns + backend env `GITHUB_INSTALL_STATE_SECRET` (§7) |
| Deployment stuck in `delete_failed` | read the error; "map of object required, but have string" = compound input stored as non-JSON; retry, else force delete (§7) |
| No **Support** entry in the sidebar / support routes 503 | backend has no `HUBSPOT_ACCESS_TOKEN`: is `hubspot-access-token-shared` (Tilt tenants: `hubspot-secret`) in the namespace and was the backend restarted after it appeared? (`devplus-platform/docs/SUPPORT_DESK.md`) |
| Support routes 502 | backend log `SupportHandler.*: HubSpot failure`: 401/403 = token/scope; `pipeline "…" not found` = `hubspot.ticketPipeline` |
| Agent reply invisible to the user | it was a HubSpot **note** (internal) not an **email** reply, or the app lacks `sales-email-read`/`crm.objects.emails.read` |

Database ops recipes: `devplus-platform/DATABASE_OPERATIONS.md`.

## 7. Runbooks

### agent-service: load, replay and the cross-language vectors
- **Load harness**: `uv run python -m agent_service.domain.evals.load --runs 30 --latency 1.0` (in `devplus-agent-service`) drives concurrent runs through the real stream with a replaying model and a synthetic tenant, entirely in memory (never a tenant DB, even in a pod), and prints durations, queued runs, worst loop lag, peak RSS and errors. Run it before any scale change.
- **Replay a live run**: set `AGENT_REPLAY_CAPTURE_DIR` on agent-service; each run writes its model replies per graph node to `<dir>/<thread>-<id>.json` (model output: a development aid, never on in production). `--script <file>` replays one under load.
- **Liveness**: `/livez` (event loop answering; up while draining) restarts a pod only after about two minutes of silence; `/health` (readiness) is 503 while draining. A loop stall over 1 s is logged ("Event loop stalled").
- **Deploy order for the harness rework**: agent-service and backend before the frontend. The SPA's reattach needs the agent's `id:` lines and the backend's `/ai/assistant/attach`; against an older agent-service a reattach replays the run from its start. The per-user run cap (`AGENT_MAX_RUNS_PER_USER`, 3) is enforced atomically: a run reserves a place in `agent_user_runs` with one conditional update before it claims its thread, so runs started at the same instant cannot exceed it, and a run back from a stall retakes a place only if one is free (else it stops). Raise it for parallel testing; the platform CI stack sets 40.
- **Cross-language vectors**: binding inference, the blank check and registry version derivation exist in Python and Go. Change a rule in every language, run `uv run python scripts/gen_cross_language_vectors.py` in `devplus-agent-service`, copy the file to `devplus-backend/services/testdata/` and `devplus-executor/services/testdata/`, and run `devplus-platform/scripts/check-shared-vectors.sh` (or `... <branch>` to compare the copies on a branch).

Support desk (HubSpot): creating the HubSpot app (developer-platform project in `devplus-platform/hubspot-app/`, static auth; classic Private Apps no longer exist on new portals), enabling per env, scopes, agent workflow, troubleshooting → `devplus-platform/docs/SUPPORT_DESK.md`.

### GitHub install-state secret: who owns what

One HMAC (`random_password.github_install_state` in core-infra, verified by the `github-callback` hub, exported as the `github_install_state_secret` output) reaches tenant backends as `GITHUB_INSTALL_STATE_SECRET` by two routes:

| Secret name | Created by | Consumed by |
|---|---|---|
| `github-install-state-shared` (key `state-secret`) | core-infra source Secret in the `external-secrets` namespace, Reflector-mirrored into every namespace matching `github_app_reflector_namespace_pattern` (default `.*`), like `github-app-secret` | the chart default (`githubInstallState.enabled=true`, `secretName=github-install-state-shared`): every tenant that sets nothing, i.e. tenants provisioned from inside DevPlus |
| `github-install-state` | `terraform-helm-devplus-services` (from core-infra remote state) and Tilt (aks: copied from the hub namespace; local/atlas: placeholder) | Terraform-managed and Tilt tenants, which override `githubInstallState.secretName` to it |

**Single-owner rule**: a Reflector mirror always keeps the source's name, and Reflector v10.0.39 adopts or overwrites a same-named non-reflection Secret, or 409-races Terraform on a brand-new namespace. Nothing else may ever create `github-install-state-shared`, and the two names must never share an owner. The secretKeyRef is deliberately required (no `optional: true`): a backend pod in a namespace the mirror has not reached yet waits in `CreateContainerConfigError` and starts by itself once it lands. Follow-up: retire the per-tenant copy and its remote-state plumbing once every tenant reads the mirror.

### Provisioned tenant cannot create GitHub connectors

- **Symptoms**: creating a GitHub connector fails with "Failed to generate install URL" (pre-fix backend: an orphan, forever-disconnected connector row was inserted first) or 500 `GitHub App install flow is not configured on this server` (fixed backend, no row inserted); `GET /api/v1/connectors/providers/github/install-url` returns the same 500. First seen on `dan.devplus.io` (prod).
- **Cause**: the tenant was provisioned from inside DevPlus (DevPlus Tenant catalog item), which sets neither `githubInstallState.enabled` nor a per-tenant `github-install-state` Secret, so the backend booted without `GITHUB_INSTALL_STATE_SECRET`. Terraform-managed and Tilt tenants were never affected.
- **Fix**: (1) apply devplus-core-infra in that environment (creates the `github-install-state-shared` source Secret; Reflector mirrors it within seconds); (2) cut a chart release carrying the new default and redeploy the tenant's DevPlus Tenant deployment with that `chart_version` (the Helm upgrade adds the secretKeyRef and rolls the backend; if the mirror has not landed yet the new pod waits in `CreateContainerConfigError` and starts by itself once it does). Interim, before a release exists: `kubectl -n <tenant> set env deployment/backend GITHUB_INSTALL_STATE_SECRET="$(kubectl -n <tenant> get secret github-install-state-shared -o jsonpath='{.data.state-secret}' | base64 -d)"`; the next Helm upgrade replaces it with the secretKeyRef.
- **Verify**: `kubectl -n <tenant> get secret github-install-state-shared`; `kubectl -n <tenant> exec deploy/backend -- sh -c 'test -n "$GITHUB_INSTALL_STATE_SECRET" && echo set'`; `GET /api/v1/connectors/providers/github/install-url` returns `{installUrl}`; then create the connector and complete the install round-trip (the connectors page shows a success or error banner from `?github_installed=1` / `?github_error=`).

### Deployment stuck in `delete_failed`

- **Read the error first**. "map of object required, but have string" means a compound-typed input (map/object/list/set/tuple) was stored as non-JSON text; the API now rejects that class with 400 at create/preview/update, and the executor names the parameter before codegen. Fix the input (Update deployment with valid JSON) or the cloud-side cause (permissions, dependents) and **retry the delete**: a plain retry re-runs the same pipeline.
- **Zero-resource short-circuit**: when the remote state is absent or holds no managed resources, the executor skips `tofu destroy`, releases the state doc, and removes the record. Deployments that were stuck with zero resources before this fix delete cleanly on a plain retry; no force needed.
- **Force delete** (last resort): the UI delete dialog's "Force delete" checkbox (deployments list and deployment page), `devplus destroy <dep> -p <proj> --force [--wait]`, MCP `delete_deployment` with `force=true` (+ `confirm`), the chat delete agent when asked explicitly, or `DELETE /api/v1/projects/{pid}/deployments/{id}?force=true`. The executor still runs a best-effort destroy (or forget for adopted deployments); on failure it logs a `FORCE DELETE:` block listing every managed address still in state (`[module.]type.name[index]`), deletes the state doc on opentofu-server, and removes the record. Force also covers a catalog item, module or secret reference that no longer loads. Two cases keep the row in `delete_failed`: a failed state `DELETE` (error "Destroy failed (...); force delete could not release the remote state (...)": check opentofu-server reachability and retry) and a state locked by another operation (drift check, refine or deploy in flight: retry once it finishes; a delete waits while a drift check is queued). Previews ignore force.
- **What gets orphaned, and where to find it**: exactly the addresses in that block. They stay in the cloud, unmanaged, and must be cleaned up by hand (or re-adopted). The block goes to the deployment's log stream (Mongo `deployments.logs` + WebSocket, i.e. the deployment page, or `devplus logs <dep> -p <proj> --follow`) and, prefixed `FORCE DELETE deployment=<id>:` with the requester, to the executor job pod log (OTel), which is the durable copy. The row with its logs is removed about 5 s after the closing line "Deployment removed by force delete; N resource(s) remain in the cloud, unmanaged"; the deployment page then shows that count on its removal screen.
- Previews ignore `force` (deleted directly). TTL expiry never forces.

### 7.x The Hermes agent (retired)

Hermes, Paperclip, their Teams bot and the `*.tools.devplus.io` hostnames were decommissioned on
2026-09-23 (workspace `CLAUDE.md` §15); the `devplus-hermes-agent` and `devplus-hermes-state`
repos are retired and archived. [HERMES_AGENT.md](HERMES_AGENT.md) is history only.
