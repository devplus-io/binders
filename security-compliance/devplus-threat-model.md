---
id: 5b2df703-e464-406a-bb4b-1c36fe691fcc
binder: binder-mvvle32a-1
folder: folder-mvw0u7cf
---
# DevPlus Threat Model

> Created 2026-10-05. Method: STRIDE per trust boundary + attack trees for the highest-value chains.
> Sources: `docs/SECURITY_MODEL.md` (§5–§7 are the honest baseline), `docs/ARCHITECTURE.md`, AGENTS.md, and targeted verification in `devplus-core-infra` (TUF root, Kyverno tfvars).
> Frame everything with the executor job-pod trust model: **a job pod is unsandboxed arbitrary code execution holding platform credentials.** Any project-write principal gets code execution in-cluster via module source / setup scripts.

## 1. Scope & assets (crown jewels)

| # | Asset | Why it's a crown jewel | Current protection |
|---|---|---|---|
| A1 | **Customer cloud credentials** (`connector_secrets`) | Assume any customer's cloud | Envelope-encrypted (AES-256-GCM, HKDF per-tenant scope), KEK from `SECRET_ENCRYPTION_KEY` |
| A2 | **oidc-service signing keys** | Tokens it mints are *trusted by AWS/Azure/GCP federation* | In-cluster only; ephemeral keys unless `oidc-keys` secret set |
| A3 | **`SECRET_ENCRYPTION_KEY` (root KEK)** | Decrypts A1 + tofu state + PAT redemption codes | Key Vault → Helm; single root key |
| A4 | **OpenTofu state** | Contains secrets (plaintext for `tfvar` expose mode; always contains provider internals) | Sealed at rest; **served with no auth** by opentofu-server |
| A5 | **`INTERNAL_API_TOKEN`** | Unlocks backend `/internal/*` incl. credential-unseal endpoints | Static shared string |
| A6 | **`WORKOS_COOKIE_PASSWORD`** | Seals sessions **and** derives the agent-run-token HMAC | Key Vault |
| A7 | **GitHub App private key** | Repo write across every installation | Key Vault / Reflector mirror into every tenant namespace |
| A8 | **HubSpot platform-wide token** | Every tenant's support tickets | Mirrored `.*` by default; tenant boundary is code (`SupportService`), not the credential |
| A9 | **`GITHUB_INSTALL_STATE_SECRET`** (hub HMAC) | Forging it spoofs GitHub-install state | Mirrored `.*` (same exposure class as A7) |
| A10 | **CI federated identities** | GitHub Actions → **Owner** on env subscriptions | Environment-scoped, repo-pattern creds, org rulesets |
| A11 | **Customer infrastructure** | The actual thing the executor deploys into | Cedar plan gate (fails open in places), governance-service (fails closed) |
| A12 | **Discovered-resource inventory** | Sensitive map of every customer's cloud | Mongo, per-tenant DB |

## 2. Trust boundaries

```
TB1  Internet ──> Cloudflare edge ──> tunnel ──> nginx/tenant-gate      (public surface)
TB2  SPA ──> backend /api/v1 + WS + /callback + /.well-known           (session boundary)
TB3  In-cluster services (oidc, opentofu, governance, agent, executor) (ClusterIP — mostly NO AUTH)
TB4  Job pods ──> platform (backend /internal, oidc, state, Mongo)     (hostile-code boundary)
TB5  Job pods ──> customer clouds                                      (federated tokens)
TB6  CI (GitHub Actions) ──> Azure subscriptions                       (federation)
TB7  Tenant ──> tenant (namespace / Mongo DB / WorkOS org)             (logical only — no NetworkPolicy)
```

**External exposure is small by design**: frontend, authed `/api/v1`, anchored `/.well-known`, `/mcp` (OAuth 2.1), `/callback`. Host guards 404 wrong Hosts; WS origin checked against the tenant host. Everything juicy is ClusterIP-only — the threat model therefore pivots on **who can get in-cluster**, and today the answer is *any project-write user via a job pod*.

## 3. STRIDE per boundary

### TB1 — Public edge
| Threat | Assessment |
|---|---|
| Spoofing | Tenant-host gating via nginx `$tenant_ok` map + backend host guard; `/healthz` and `/.well-known` exempt (info-leak check: what exactly does `.well-known` serve?) |
| Tampering | WAF only in **prod**; dev/stage ride Cloudflare without WAF policy |
| Info disclosure | 5xx never echo internals; `safelog` strips CR/LF. Verify: error shapes on `/api/v1`, HubSpot 503s, `/mcp` OAuth metadata |
| DoS | Edge-protected; but AI/deploy endpoints are expensive — no rate limiting documented |
| Elevation | WorkOS callback is centralized (`auth.<tier>/callback` → org→instanceUrl mapping); test open-redirect / cross-tenant round-trip |

### TB2 — SPA ↔ backend (session)
| Threat | Assessment |
|---|---|
| Spoofing | Sealed httpOnly cookie (AES-256-GCM); role always from Mongo, never claims (good). XSS can still act as the user — SPA is the XSS surface |
| Tampering | CSRF = `X-DevPlus-Csrf: 1` presence on mutations — sound **iff** CORS never reflects arbitrary origins with credentials; **verify**. WS: origin checked against tenant host (good) |
| Repudiation | Agent actions carry the user's identity (run token "acts exactly as the user") — attribution is preserved; audit trail completeness unverified |
| Info disclosure | IDOR is the classic risk: Cedar guards routes; per-object project/tenant scoping is **code, not credential** (HubSpot precedent). Prime black-box target |
| DoS | Long-running deploys/AI runs; no per-tenant quotas documented |

### TB3 — In-cluster service mesh (the big one)
`docs/SECURITY_MODEL.md` §5 states it plainly — these are **unauthenticated**, bounded only by ClusterIP reachability:

| Surface | Auth | Impact if reachable |
|---|---|---|
| opentofu-server — registry + **all state** | none | Plaintext secrets out of any tenant's state (A4) |
| oidc-service `/token/project`, `/token/deployment` | none | **Mints federation-grade tokens, caller-overridable `sub`** (A2) |
| governance `/evaluate`, `/reload` | none | Policy oracle; `/reload` reads Mongo (tamper requires Mongo write) |
| agent-service | per-run token only | Agent principal = user's identity |
| executor → agent `/refine-module` | none | Module-content injection into a deploy path |
| backend `/internal/*` | static `X-Internal-Token` (A5) | Includes credential-unseal endpoints |

With **no NetworkPolicy** (§7 "the big one") and job pods running user-supplied code, TB3+TB4 collapse into one reachable domain. See Attack Tree 1.

### TB4 — Job pods (hostile code in-cluster)
Job pod holds: cloud federated creds (intended), Mongo per-tenant user, and — to verify — `INTERNAL_API_TOKEN` if it calls `/internal/connectors/{id}/gcp-credentials`. Every secret mirrored with pattern `.*` (A7, A8, A9) is readable. Pod hardening (uid 65532, read-only rootfs, TMPDIR) is good but irrelevant to network reach.

### TB5 — Job pods → customer clouds
Per-connector scoping is layered correctly (WIF attribute conditions, role/SA bindings, permission-RANK-negotiated GitHub tokens). The boundary holds **as long as TB3/TB4 don't let a pod mint or fetch someone else's token**.

### TB6 — CI → Azure
`devplus-cicd-infra` grants **Owner** on env subscriptions. Controls: repo-pattern federated creds, environment protection, org rulesets + Actions allowlist, dependency-review. Blast radius of a single compromised workflow = whole subscription (A10). Live-drift risk in `devplus-github-settings` ruleset is documented.

### TB7 — Tenant ↔ tenant
tenant = namespace = Mongo DB = WorkOS org. Mongo DB users scoped to own DB (`atlasAdmin` bug fixed — verify via pod restart). But in-cluster, the *network* is flat: tenant isolation today is enforced by credentials (per-DB Mongo users, Cedar), **not** by networking. Any credential leak from a mirrored secret (A7–A9) or the flat ClusterIP domain punctures it.

## 4. Attack trees

### Tree 1 — Cross-tenant cloud takeover from a single module (CRITICAL)
```
Goal: read tenant B's cloud using tenant A's project-write access
A. Get code execution in a job pod                        [legit feature: module source / setup scripts]
   └─ A1. Module contains a curl/terraform data source that exfiltrates or probes
B. From the pod (no NetworkPolicy → whole ClusterIP domain reachable)
   ├─ B1. POST oidc-service /token/deployment, choose sub + aud
   │       → AssumeRoleWithWebIdentity on ANY connector's role (slug-guessable: DevPlus-<slug>)
   ├─ B2. GET opentofu-server state for tenant B  → plaintext secrets (tfvar mode) (A4)
   ├─ B3. GET backend /internal/connectors/{id}/gcp-credentials  → unsealed creds (A5, if pod holds the token)
   └─ B4. Read mirrored secrets: GitHub App key, HubSpot token, install-state HMAC (A7–A9)
```
No exploit, no vuln, no misconfiguration — **every edge is intended behavior**. The docs already name the compensating frame: "real boundaries are network + identity scoping", and neither exists between pods and TB3 today.

### Tree 2 — oidc-service = keys to every cloud
```
Compromise oidc-service (or its oidc-keys secret) → mint tokens with chosen sub/aud
  → AWS AssumeRoleWithWebIdentity / Azure client_assertion / GCP STS for ALL tenants
```
Attackers' route today is Tree 1-B1 (unauthenticated mint); direct compromise needs in-cluster RCE. The overridable `sub` is *by design* (AWS aud/sub trust conditions) — which is exactly why the endpoint cannot be exposed beyond ClusterIP.

### Tree 3 — Poison the supply chain
```
A. Registry: artifacts mutable, overwrite-in-place, no checksums/signatures
   → overwrite a catalog module → next deploy of any project using it runs attacker code (→ Tree 1 pod)
B. Images: Kyverno verify-ghcr-images is **Audit in ALL envs** (verified in tfvars 2026-10-05)
   and the pinned GitHub TUF root v8 **expired 2026-08-01** → the attestation gate is likely a silent no-op
C. CI: dependency-review blocks moderate+ on PRs only; direct pushes to main in a service repo
   dispatch `service-image` into platform CI (hermetic tests mitigate)
```

### Tree 4 — Prompt injection → destructive action
```
Malicious content in indexed corpus / discovery output / repo content → injected instruction
  → agent calls deploy/delete tool with user's identity
Backstops: Cedar double-eval (agent principal + user token); plan gate.
Fail-open holes: backend plan gate fails open with no governance client;
executor PLAN_GATE_FAIL_MODE defaults open on transport errors.
Force-delete is a deliberate orphaning primitive (state released, cloud resources abandoned,
only evidence is a log block that dies with the row).
```

### Tree 5 — CI → subscription
```
Compromised workflow / allowlisted action → OIDC token → env-scoped federation → Owner on subscription
```
Mitigated by environment scoping + org controls; residual risk is a poisoned allowlisted dependency (cf. xz-class).

## 5. Risk register

| ID | Risk | L | I | Status / control | Gap |
|---|---|---|---|---|---|
| R1 | Job pod → flat ClusterIP → federation-token mint / state read / internal unseal (Tree 1) | M | **Critical** | **None** — documented accepted risk | NetworkPolicy between tenant workloads + egress policy on job pods |
| R2 | opentofu-server: unauthenticated global read of **all** tenant state | M | Critical | ClusterIP only | Per-tenant auth on state API; or same fix as R1 |
| R3 | oidc-service unauthenticated mint with caller-chosen `sub` | M | Critical | ClusterIP only | Same as R1; long-term: bind sub to caller identity |
| R4 | Single root KEK (`SECRET_ENCRYPTION_KEY`) decrypts every tenant's secrets + state | L | Critical | Key Vault, rotation path exists (`SECRET_ENCRYPTION_KEYS`) | Per-env keys exist? Rotation drill? |
| R5 | Image-attestation gate is a silent no-op (Audit everywhere + expired TUF root v8) | **H** (verified) | High | Audit mode reports violations | Re-pin TUF root, verify client follows root rotation, re-enable Enforce after sigstore config fix |
| R6 | Module registry: mutable, unsigned, no checksums | M | High | none | Content-addressed or signed artifacts |
| R7 | Mirrored secrets (`.*` pattern): GitHub App key, HubSpot token, install-state HMAC readable by any workload | M | Medium-High | Tenant boundary is code | Narrow reflector patterns; hub-proxy the HubSpot token |
| R8 | Prompt injection → destructive cloud action; plan gate fails open (2 places) | M | High | Cedar double-eval; governance fails closed | Set `PLAN_GATE_FAIL_MODE=closed`; wire backend plan gate to always have a client |
| R9 | CI identity = Owner on subscription | L | Critical | env-scoped, repo-pattern | Split roles; PIM/just-in-time for prod |
| R10 | `/internal/*` protected by one static shared token, incl. credential unseal | M | High | static token | mTLS/service-account auth; confirm job pods don't carry it |
| R11 | HubSpot token = every tenant's tickets | M | Medium | `SupportService` filter + requester check (unit-tested) | Narrow mirror pattern |
| R12 | Governance logs full evaluation payloads | H | Low-Med | — | Redact payloads |
| R13 | oidc `/token/validate` HMAC fallback; missing-`aud` treated as pass; non-constant-time compare | M | Low-Med | documented | Tighten in next oidc bump |
| R14 | No state-lock TTL; crashed run ⇒ manual unlock | M | Low | runbook | TTL on locks |
| R15 | Force-delete orphans cloud resources with no durable record | M | Med | deliberate feature | Append-only record of orphaned resources |
| R16 | No WAF in dev/stage; no rate limits on AI/deploy endpoints | M | Low-Med | Cloudflare edge | Rate limiting on expensive routes |
| R17 | devplus-github-settings live ruleset drift | H (documented) | Med | README flags it | Reconcile before any apply |

## 6. Verified findings (this session, 2026-10-05)

1. **TUF root expiry is real**: `devplus-core-infra/modules/image-verification/main.tf` pins GitHub TUF root v8, comment: "expires 2026-08-01". TUF clients can chain to a rotated root only if rotation happened before expiry and the client follows it — verify in-cluster (Policy Reports for `verify-ghcr-images`).
2. **Kyverno gate is Audit in dev, stage AND prod** — `environments/{dev,stage,prod}.tfvars` all set `image_verification_failure_action = "Audit"`. The comment in `bootstrap.tf` ("stage/prod Enforce") is stale.
3. Net effect: **nothing currently blocks an unattested `ghcr.io/devplusinc/*` image**, and violations may not even be reported if the TUF chain is broken.

## 7. What the black-box pentest should target (external, no code)

Ordered by expected yield:

1. **Auth flows**: WorkOS callback (`auth.dev.devplus.io`) — redirect handling, org→instanceUrl mapping, session cookie flags, sealed-cookie integrity.
2. **CSRF/CORS**: mutation without `X-DevPlus-Csrf`; CORS reflection on any `/api/v1` route; WS handshake cross-origin.
3. **IDOR across tenants/projects** (two throwaway tenants): project IDs, connector IDs, support desk, github connector routes (`/api/v1/github/connectors/{id}/...`).
4. **Surface audit**: confirm only frontend + `/api/v1` + `/.well-known` + `/mcp` + `/callback` answer; probe opentofu-server/oidc/agent/8081-8085 externally (should 404/timeout).
5. **github-callback hub**: state HMAC forgery, `tenant_hostname_pattern` bypass (dot-names, casing, trailing chars), `%20` handling.
6. **devplus-mcp** OAuth 2.1: metadata, token audience, scopes.
7. **Injection surfaces**: module-source fields, catalog parameter bindings (`tfvar` expose mode!), support-desk fields, AI chat → tool args.
8. **Error/log hygiene**: 5xx shapes, HubSpot 503s, header injection in tenant-host-dependent responses.
9. **Rate limiting / DoS-by-cost** on AI + deploy triggers.
10. **Host header games** on `<tenant>.devplus.io` vs `app.devplus.io` vs bare `*.devplus.io` wildcards.

Internal-surface items (R1–R3, R10, R5-in-cluster) need an in-cluster position — run those as a second phase from a disposable tenant's job pod.

## 8. Assumptions & out of scope

- Prod tenants' real data untouched; all testing against disposable tenants.
- No DoS, no WARP/Cloudflare infra attacks, no Atlas/Azure control-plane attacks outside the ROE.
- Hermes/Paperclip are decommissioned (§15 AGENTS.md) — excluded.
- This model assumes docs match `origin/main`; deviations found during testing get corrected in both docs and the Binders knowledge base.

## 9. Deep code audit addendum (2026-10-05)

A full code-level audit (see the SECURITY_AUDIT_FINDINGS note) verified the register first-hand. Register status: **R1–R6, R10, R12, R16 all CONFIRMED in code**; R1 is **worse than modeled** (job pods also hold `GITHUB_APP_PRIVATE_KEY`, and the oidc mint endpoint allows claim *override*, not just caller-chosen sub). Three findings change the picture:

- **NEW CRITICAL [TILT-AKS]:** the repo-committed `JWT_SECRET` (`default-secret-change-in-production`, Tiltfile:579) + the oidc HMAC fallback + the backend's missing-`aud` pass = **unauthenticated full backend compromise over the public internet** for Tilt-managed AKS tenants. Trees 2/5 are reachable from the *internet*, not just in-cluster.
- **NEW HIGH:** backend deployment IDOR (`/projects/{projectId}/deployments/{id}` never binds id to projectId — read/update/delete/log-injection). Cedar evaluates path strings and cannot catch it.
- **NEW HIGH:** agent-service interrupt resume has no ownership check (approval-card hijack + transcript leak).
- **REFUTED (crypto):** cross-scope state decrypt — HKDF salt + GCM AAD correctly bind the tenant scope; cross-tenant state access is purely network-level (still real).
- **REFUTED (system-level):** "governance fails closed" — the service does, the pipeline does not (executor fail-open default, nil-client allow, silent context degradation, Mongo-error reload ⇒ deny-all kill-switch).
