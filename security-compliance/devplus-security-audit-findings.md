---
id: a9ee5f71-b433-47db-b13f-90bb750f9330
binder: binder-mvvle32a-1
folder: folder-mvw0u7cf
---
# DevPlus Security Audit — Findings

> 2026-10-05. Full code-level audit of all security-critical services. Method: six parallel deep-audit passes (backend, oidc-service, opentofu-server, executor, governance+agent, platform/edge/infra) with a mandatory verification checklist; every Critical/High finding below was **independently verified by reading the code** (file:line cited). Companion to the THREAT_MODEL note.
>
> **Scope note:** Terraform-managed tenants (per-tenant random secrets) and Tilt-managed tenants (repo-literal secrets) have materially different exposure. Findings marked **[TILT-AKS]** apply to internet-exposed Tilt-managed tenants.

---

## 0. Executive summary

The perimeter is well-built. The interior is not, and one committed secret turns an interior bug into an internet-reachable full compromise for Tilt-managed tenants.

**The worst chain (C1 + C2 + F1, all verified):**

```
Repo contains literal "default-secret-change-in-production" (Tiltfile:579)
  → applied as jwt-secret in ALL Tilt modes INCLUDING aks
  → oidc-service /token/validate HMAC fallback accepts any JWT signed with it
  → attacker forges HS256 {"role":"system"} (no aud) — fully offline
  → sends as Bearer to https://<tenant>.dev.devplus.io/api/v1/* (PUBLIC INTERNET)
  → backend verifyOIDCToken → oidc /token/validate → HMAC fallback VALID
  → missing aud passes audience check (backend middleware.go:462-465)
  → isSystemOIDC: role=="system" → full admin API + /internal/* access
  → /internal/connectors/{anyId}/(aws|azure|gcp)-credentials → every tenant cloud cred
```

No exploit code runs on the target, no auth is bypassed in a classic sense — the system *validates* a token the attacker made with a key published in the repo.

---

## 1. Critical

### C1 — Unauthenticated federation-token minting with caller-overridden claims (verified)
**Where:** `devplus-oidc-service/internal/httpapi/handlers.go:49-118`, `internal/token/service.go:88-90, 110-112`
`/token/project` and `/token/deployment` have **no auth of any kind**. The `additionalClaims` override loop runs **after** base claims, so the caller overwrites `sub`, `aud`, `iss`, `exp`, `role`. Any cluster pod can mint an RS256 token satisfying a connector's IAM trust policy (`claims.sub = "devplus:<slug>"` or aud-only for legacy) → `AssumeRoleWithWebIdentity` → **cross-tenant cloud takeover**. Also: `{"audience":"backend","claims":{"role":"system"}}` → backend `internal@system` identity. `exp: 9999999999` defeats the 1h TTL. oidc-service is stateless, so nothing can validate the deployment/connector binding.
**Fix:** require internal auth on both mint endpoints; strip reserved claims (`iss/exp/iat/nbf/sub/aud/role/scope`) from caller override; bind sub to a server-resolved deployment→connector mapping.

### C2 — HMAC fallback in /token/validate; JWT_SECRET defaults to a public constant (verified)
**Where:** `internal/token/service.go:133-138`, `internal/config/config.go:26-30`, backend `handlers/middleware.go:235, 456-492, 462-465`
The validate endpoint accepts **any HMAC-signed token** against `JWT_SECRET`, whose default is the literal string `"default-secret-change-in-production"` **published in the repo**. Missing `aud` passes the backend's audience check. See the chain in §0.
**Fix:** delete the HMAC fallback (migrate any consumer first); fail boot if `JWT_SECRET` unset; require `aud` in `audienceAllowed`.

### C3 — opentofu-server: every route unauthenticated; all tenants' decrypted state readable cluster-wide (verified)
**Where:** `internal/server/routes.go:14-36` (zero auth middleware; confirmed by reading the full router), `services/state_service.go:71-105` (decrypts per-request)
Any cluster pod can `GET /state/{project}/{name}` on any tenant's opentofu-server ClusterIP and receive **decrypted** state (tfvar-mode secrets are plaintext by design; envelope encryption is transparent to the caller because the service unseals). Also unauthenticated: module upload/overwrite, `DELETE /state` (ignores locks), LOCK/UNLOCK.
**Fix:** auth on state+upload routes (tofu HTTP backend supports custom headers via backend.hcl); default-deny NetworkPolicy; treat "ClusterIP" as "cluster-wide" until then.

### C4 — Job pods are unsandboxed ACE holding platform-wide credentials (verified)
**Where:** `devplus-executor/services/job_dispatcher.go:683-872` (`buildJobSpec`)
Deployment/module/adoption job pods hold in env: `INTERNAL_API_TOKEN`, `MONGODB_URI` (tenant DB creds), **`GITHUB_APP_PRIVATE_KEY` (org-wide GitHub write!)**, `INFRACOST_API_KEY`, plus a **mounted K8s SA token** (no `automountServiceAccountToken: false`), **no securityContext at all**, and an optional **hostPath** mount (`:794-809`) — a node-escape primitive. Setup scripts inherit the full pod env (`opentofu_executor.go:862`). Any project-write user's module source or setup script gets all of it. The repo-analysis pod (`:1198-1343`) proves the team knows how to harden (uid 65532, read-only rootfs, no SA token, no App key) — that profile just wasn't applied to deploy pods.
**Fix:** strip `GITHUB_APP_PRIVATE_KEY` from job pods (mint scoped tokens backend-side instead, as repo-analysis already does); drop INTERNAL_API_TOKEN for deployment-bound service tokens; apply the repo-analysis securityContext profile; remove/replace hostPath.

### C5 — [TILT-AKS] Committed secrets applied to internet-exposed tenants (verified)
**Where:** `devplus-platform/Tiltfile:576-593`
`jwt-secret` = `default-secret-change-in-production` and `secret-encryption-key` = base64("devplus-dev-state-encryption-key") are created **unconditionally — including `use_aks`** — and consumed by oidc-service, backend, and opentofu-server. Anyone with repo read access can forge system tokens (C2) and knows the KEK for connector secrets + tofu state on every Tilt-managed AKS tenant (e.g. `helder.dev.devplus.io`). Terraform-managed tenants are safe (`locals.tf:6,10` per-tenant random).
**Fix:** gate both on `not use_aks`; source aks values from Key Vault (as `github-install-state` already does); **rotate** on all Tilt-managed tenants.

---

## 2. High

### H1 — Cross-project deployment IDOR: read, update, DELETE, log-injection (verified)
**Where:** `devplus-backend/handlers/deployment_handler.go:332-352` (GetByID), `:535-596` (Delete), `:650-777` (Update), `:780-796` (BroadcastLog), `websocket_handler.go:122-131`
Routes are `/projects/{projectId}/deployments/{id}`, but handlers fetch by ID **only** — no `deployment.ProjectID == projectId` check. Cedar evaluates the *path string*, so it cannot catch this. A user authorized on project A can read (logs + **inputs**, which carry secret values), modify, **destroy-trigger** (Delete → executor destroy of project B's cloud infra), and stream/inject fake logs into project B's deployments within the tenant. `BroadcastLog` and `CatalogHandler.BroadcastLog` check nothing at all.
**Fix:** one-line ownership predicate (`deployment.ProjectID == path projectId`) in all five sites; same for `ProjectHandler.GetByID`.

### H2 — Agent-service interrupt resume: no ownership check → approval hijack + transcript leak (verified)
**Where:** `devplus-agent-service/src/agent_service/api/utils/stream.py:193-216` (`_claim_thread` checks lease/liveness only, never ownership), `routers/chat.py:266`, `:588-601` (identity optional — "No credential, no identity: … the run proceeds")
`POST /chat/stream` with a victim's `thread_id` + `approval` resumes **another user's pending interrupt**: the attacker answers the victim's approval card (`provision_catalog_item` takes parameters/deploymentName from the resume payload) and receives the victim thread's streamed output. `GET /chat/threads/{id}/state` *does* enforce ownership — the resume path is the inconsistent one.
**Fix:** enforce thread ownership in `_claim_thread`/resume; require a credential on resume.

### H3 — Plan gate fails OPEN, three ways (verified twice, independently)
**Where:** `devplus-executor/services/executor_service.go:995-1006` (default = proceed unless `PLAN_GATE_FAIL_MODE=closed`), `:1368-1374` (adoption path same), `devplus-backend/handlers/internal_handler.go:421-424` (nil governance client → `Allow: true` — currently unreachable but wrong), plus **silent context degradation**: cost-estimate failure ⇒ `totalMonthlyCost` omitted (`internal_handler.go:449-453`), resource-action extraction failure ⇒ `hasResourceReplace=false, count=0` (`:457-475`) — cost- and replace-conditioned Cedar policies silently stop binding.
**Fix:** default `PLAN_GATE_FAIL_MODE=closed` in the chart; nil client → 503; fail the deployment when extraction fails; log the degradation.

### H4 — Module registry: unauthenticated publish, mutable versions, no checksums → supply-chain RCE
**Where:** `devplus-opentofu-server/internal/server/routes.go:22-23`, `services/registry_service.go:102-107` (delete-then-overwrite same version), no integrity on push or pull
Poisoning a module version is unauthenticated and silent; the poisoned `.tf` executes in a job pod (see C4) with cloud creds.
**Fix:** auth on upload/delete; immutable versions (reject overwrite); sha256 in the download response.

### H5 — Governance-service fully unauthenticated; /reload is a deny-all kill-switch
**Where:** `internal/server/routes.go:15-21` (no auth), `services/governance_service.go:63-66` (Mongo error during reload ⇒ **empty policy set swapped in** ⇒ deny-all until next success), `handlers/middleware.go:9` (CORS `*`)
Any pod can use `/evaluate` as an authz oracle, enumerate policy names via `/policies/loaded`, DoS via unthrottled `/reload`, or — during a Mongo blip — flip the tenant to deny-all (or, combined with H3's fail-open executor, to allow-all).
**Fix:** internal-token auth on all routes; rate-limit `/reload`; **fail reload on Mongo error (keep last-good set)** — this one is a small change with big payoff.

---

## 3. Medium

| # | Finding | Where (evidence) |
|---|---|---|
| M1 | `INTERNAL_API_TOKEN` is a shared static token with **no binding**: any job pod can unseal **any** connector's AWS/Azure/GCP credentials by ID (contrast: GitHub mint is lease-bound via `GitHubMintGuard` 409 check) | `devplus-backend/handlers/internal_handler.go:155, 221-392`; executor `gcp_credentials_service.go:141` |
| M2 | Missing `aud` passes the backend audience check (OIDC fallback branch) | `middleware.go:462-465` |
| M3 | Project-restricted PATs bypass the allowlist on routes without `{projectId}`: tenant-wide `/deployments`, `/deployments/cost-history`, `/search`, `/agent-actions` | `middleware.go:335-353` |
| M4 | Authenticated log injection: `BroadcastLog` (deployment + catalog) has no ownership check | `deployment_handler.go:780-796`, `catalog_handler.go:347-362` |
| M5 | `/token/service` caller-chosen `audience` → allowed pods can target legacy aud-only connector trust policies | `oidc handlers.go:151-154`; `aws.go:585-587` |
| M6 | PR-triggered platform workflows read Key Vault (GitHub App id + private key) with a repo-wide federated identity — same-repo-PR workflow edits exfiltrate; fork PRs fail closed | `ci.yml:34-46`, `platform-test-matrix.yml:98-110` |
| M7 | Shell injection via workflow inputs on a secret-laden runner: `${{ inputs.chart_version }}` / `inputs.environment` / `inputs.version` unquoted in `tofu`/`sh` | `_deploy-environment.yml:141-149`, `release.yml:179` |
| M8 | Chart-wide: no securityContext anywhere (frontend nginx runs as **root**), no NetworkPolicy resources exist | grep across chart; `frontend.yaml` |
| M9 | `admin_seed` null → chart defaults `admin@devplus.io`/`admin123` | `terraform-helm-devplus-services/variables.tf:175-184` |
| M10 | Governance + backend log **full evaluation payloads and entities JSON** (roles, groups, project attributes) at INFO | `governance_handler.go:32-35`, `governance_eval.go:40-50`, `governance_client.go:45` |
| M11 | Kyverno gate: Audit in all envs (verified in all 3 tfvars) + pinned TUF root v8 **expired 2026-08-01** (today 2026-10-05) — rotation chains only while the mirror is reachable; fail-open (`failurePolicy: Ignore`) | `image-verification/main.tf:26-38`, `verify-images.yaml:15`, `environments/*.tfvars` |
| M12 | State locking defects: no TTL (`Created` stored, never checked); **lock ID disclosed in the 423 body** (bypassable); writes without `?ID=` skip verification entirely | `state_service.go:168-175`, `state_handler.go:63-70, 130-135` |
| M13 | Legacy plaintext state served as-is until next write; stays plaintext in Mongo indefinitely if never rewritten | `state_service.go:100-104` |
| M14 | Executor logs unredacted: `RedactSecrets` exists but is not applied on `appendLog`/`runWithLogs` — setup-script or provider output (may echo env secrets) stored + broadcast | `executor_service.go:754-767`, `opentofu_executor.go:1698-1744` |
| M15 | Image-attestation policy is the only pod admission control and it's fail-open; no namespace/SA exclusions audited beyond Kyverno defaults | `verify-images.yaml` |
| M16 | Reflector patterns default `.*` for `github-app-secret`, `github-install-state-shared`, `hubspot-access-token-shared` — every namespace (job pods included) can read the GitHub App key and the platform-wide HubSpot token | `core-infra/variables.tf:202-206, 318-322` |

---

## 4. Low / Info (condensed)

- `err.Error()` echoed in 404 `details` (deployment/project handlers) — driver errors leak; `RespondWithReadError` exists but isn't used there.
- Logout is client-side only (no server-side invalidation of sealed cookie).
- WS: empty `Origin` always allowed; `TENANT_HOSTNAME` unset → allow-all (local-only config).
- Host guard: empty Host passes (HTTP/1.0); backend `/health` and nginx `/api/health` ungated.
- nginx: **no security headers at all** (HSTS/CSP/X-Content-Type-Options); `gzip` on authenticated JSON; no-Host requests bypass the tenant gate.
- `/token/user` non-constant-time compare (`handlers.go:183`); discovery doc advertises nonexistent endpoints; RS384/RS512 accepted in validate's RSA branch.
- oidc-service: no `Cache-Control` on JWKS; multi-replica key divergence prevented only by `replicas: 1`.
- opentofu-server: unbounded request bodies vs 256Mi pod (memory DoS); OTel traces record full request paths.
- Agent-service: CORS `*`+credentials; `/chat`+`/chat/stream` run the graph with **no credential** (unauthenticated LLM burn); `/refine-module` accepts no credential at all; pre-index threads readable cross-user (TTL-bounded).
- `totalMonthlyCost` float64→Long truncation in Cedar conversion (edge-case policy bypass).
- CI: `chart_version` has no minimum floor; skip paths (`plan_only`, `quality_gate: never`) documented by design; scripts have arg-injection footguns (developer-supplied args only).
- Local-mode job pods ride the dispatcher SA (jobs create/delete); control-plane tenants give job pods **cluster-admin** (gated, documented).
- Executor: blind SSRF via attacker-influenced `git` URLs; `AWS legacy` creds read directly from Mongo connector docs.

---

## 5. Threat-model claims — final verdicts

| Claim | Verdict |
|---|---|
| oidc `/token/project` + `/token/deployment` unauthenticated; caller-chosen `sub`/`aud` | **CONFIRMED** (read directly) |
| `/token/validate` HMAC fallback; `JWT_SECRET` defaults to a public constant | **CONFIRMED** (read directly) |
| Backend: missing `aud` passes audience check | **CONFIRMED** (read directly) |
| Backend deployment IDOR (read/update/delete/WS) | **CONFIRMED** (read directly) |
| Job pod receives `INTERNAL_API_TOKEN` (+ App key, Mongo URI) | **CONFIRMED** — worse than modeled: `GITHUB_APP_PRIVATE_KEY` too |
| Plan gate fails open by default | **CONFIRMED** — and the backend adds a nil-client `Allow:true` |
| opentofu-server state API unauthenticated; cross-tenant readable | **CONFIRMED** (crypto cross-scope decrypt itself REFUTED — HKDF salt + GCM AAD bind tenant scope correctly; the bypass is purely network-level) |
| Registry artifacts mutable, no checksums | **CONFIRMED** |
| Governance fails closed | **REFUTED at system level** — service alone fails closed; the *pipeline* fails open (executor default + nil-client path + silent context degradation + Mongo-error reload ⇒ empty set) |
| Governance endpoints unauthenticated incl. `/reload` | **CONFIRMED** (reload body ignored — poisoning requires Mongo write; DoS/kill-switch confirmed) |
| Agent tools double-evaluated in Cedar | **CONFIRMED** (plus human interrupt() approval on destructive tools) |
| Plan-gate context attacker-zeroable | **PARTIALLY CONFIRMED** — not directly by external attackers; failure paths silently zero it; internal-token holders can zero it arbitrarily |
| nginx tenant gating | **CONFIRMED**, one nuance: no-Host requests bypass; `/api/health` also ungated |
| Edge exposes only frontend + /api/v1 + anchored .well-known + /mcp + /callback | **CONFIRMED** (nginx + Cloudflare tunnel configs both checked) |
| github-callback hub HMAC/anti-open-redirect | **CONFIRMED OK** — timing-safe, anchored pattern enforced at boot, scheme/path fixed |
| Reflector `.*` patterns | **CONFIRMED** |
| TUF root expired + Audit everywhere | **CONFIRMED** |
| CI `service-image` dispatch is injection-safe | **CONFIRMED** (no `pull_request_target` anywhere) |

---

## 6. What held up (verified clean)

Session sealing (AEAD + fresh nonce + HKDF, correct flags) · CSRF coverage (no missed mutation route, sound presence-only design behind SameSite=Lax) · CORS (no origin reflection, no credentials header) · Host guard (every attempted bypass fails closed) · Support desk (tenant filter + requester check on **every** read; generic 404s) · Repo-analysis scoping (explicit anti-IDOR, 404-no-leak, lease-bound scoped GitHub tokens with echo verification) · PAT hashing (SHA-256 of ~190-bit secrets — correct choice) · Agent run tokens (separate HKDF label, constant-time, project-bound) · Provider responses redacted · CR/LF log-injection defenses · Stored-module tar extraction & symlink defense · `env_tfvar` secret mode design · TokenReview-gated `/token/service` · Repo-analysis pod hardening (test-asserted) · github-callback hub implementation.

---

## 7. Remediation roadmap (ordered by leverage)

**P0 — this week (internet-exploitable):**
1. Rotate `JWT_SECRET` + `SECRET_ENCRYPTION_KEY` on all Tilt-managed tenants; gate the Tiltfile literals on `not use_aks` (C5, C2).
2. Delete the oidc HMAC fallback (or fail boot on unset `JWT_SECRET`) (C2).
3. Add the deployment↔project ownership predicate in the five backend handlers (H1) — one-line each.

**P1 — next sprint (kills the R1 kill-chain):**
4. Auth on oidc mint endpoints + strip reserved claims from override (C1).
5. Auth on opentofu state/upload routes (C3) — backend.hcl supports headers.
6. Strip `GITHUB_APP_PRIVATE_KEY` + `INTERNAL_API_TOKEN` from job-pod env; use deployment-bound service tokens + backend-minted scoped GitHub tokens (C4).
7. `PLAN_GATE_FAIL_MODE=closed` default + nil-client 503 + fail-on-extraction-error (H3).
8. Default-deny NetworkPolicy per tenant namespace; allow-list backend/oidc/opentofu/executor/mongo only.

**P2 — hardening:**
9. Governance: internal-token auth, keep-last-good on reload error, rate-limit (H5). Registry: immutable versions + checksums (H4). Agent resume ownership (H2). PAT allowlist on global routes (M3). securityContext chart-wide (M8). TUF re-pin + Enforce after config fix (M11). Re-pin → verify → enforce. Reflector patterns narrowed (M16). Admin seed required (M9). Workflow input quoting (M7). RedactSecrets on deployment logs (M14).

---

## 8. Updated score

| Vantage | Before | After (audit) |
|---|---|---|
| External / black-box | 8/10 | **5.5/10 for Tilt-managed tenants** (C2+C5 chain is internet-reachable); **7.5/10 for Terraform-managed tenants** (per-tenant secrets) |
| Insider / job-pod | 3.5/10 | **3/10** (C1+C3+C4 verified in code; kill-chain is fully concrete, not hypothetical) |
| Ops maturity | 7/10 | 6/10 (TUF expiry + stale comments + committed secrets show drift is not being caught by any automated check) |
| **Overall** | 6.5/10 | **5.5/10** — P0+P1 items would restore to ~8.5 |

*Method note: six parallel audit passes over the security-critical services (backend, oidc-service, opentofu-server, executor, governance+agent, platform/edge/infra), each with a verification checklist; every Critical/High item verified by direct code read (cited file:line). Coverage is complete for auth/session/token/state/credential paths; non-security code paths (business logic, UI) were sampled, not exhaustively read.*
