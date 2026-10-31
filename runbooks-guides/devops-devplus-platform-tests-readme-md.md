---
id: f50d0658-7be8-451c-b860-1e32d2c01e1e
binder: binder-mvvle32a-1
folder: folder-mvw0u6xp
---
DevOps | devplus-platform/tests/README.md

Source: DevPlusInc/devplus-platform/tests/README.md, branch docs/devops-refresh-2026-10, refreshed 2026-10-04

Refreshed 2026-10-04 against origin/main code (DevPlusInc/devplus-platform, tests/README.md).

# DevPlus Platform Test Suite

Black-box testing for the DevPlus platform itself: the seven runtime services,
the OpenTofu state/registry protocols, Cedar governance, OIDC federation, the
executor dispatcher and the React SPA. `devplus-cli` and `devplus-mcp` are out
of scope and have their own suites.

Three pillars, one environment contract, one command in CI:

| Pillar | What | Where | Budget |
|---|---|---|---|
| 1 Smoke | 7-service liveness, every backend route family and every operation mounted, non-destructive lifecycle ping | `cmd/smoke` (Go) | < 60 s |
| 2a Contract | OpenAPI 3.1 harness over every operation in the backend's spec (207 at 0.1.16): 401 sweep, Cedar 403 sweep, secret-redaction invariant, response-schema conformity, coverage ledger | `contract/` (Go test) | ~5 min |
| 2b Regression | Deep stateful journeys: deployment lifecycle, state-lock contention, WebSocket logs, AI SSE + action-card interrupt, OIDC TokenReview federation, Cedar hot-reload, executor dispatch, API keys, identity, secrets, RAG | `regression/` (Go test) | ~10 min |
| 2c E2E | Browser journey: WorkOS login → catalog → deployment form → live log terminal → AI action card | `e2e/` (Playwright) | ~5 min |
| 3 Performance | k6: read-heavy (300 VUs), mutation + lock contention (50+50 VUs), streaming (100 WS + 30 SSE) with `p(95) < 200 ms` / 0 % error thresholds | `perf/` (k6) | ~4 min |

## Layout

```
tests/
├── go.mod                       module github.com/DevPlusInc/devplus-platform/tests
├── internal/
│   ├── env/                     one environment contract for every suite (+ .seed.env loader)
│   ├── httpx/                   JSON client with PAT auth, capture, Poll
│   ├── mongo/                   direct MongoDB access for queue/lease/policy assertions
│   ├── openapi/                 spec loader, operation inventory, response-schema validator
│   └── jwks/                    JWKS fetch + RS256 verification
├── cmd/
│   ├── seed/                    bootstraps users, PATs, project, connector, module, template → .seed.env
│   └── smoke/                   Pillar 1 probe runner (table or -json)
├── contract/                    Pillar 2a — spec-driven harness + expectation tables
├── regression/                  Pillar 2b — one journey per file
├── e2e/                         Pillar 2c — Playwright (fixtures/platform.ts does the real login)
├── perf/                        Pillar 3 — k6 (lib/config.js shared)
├── mocks/                       hermetic stand-ins: mock-workos, mock-aoai, mock-k8s (+ Dockerfile)
└── ci/
    ├── docker-compose.yml       the whole platform + mocks, ports published like Tilt
    └── run-suite.sh             wait → seed → smoke → contract → regression → e2e → perf
```

CI entry points: [`.github/workflows/platform-test-matrix.yml`](../.github/workflows/platform-test-matrix.yml)
(hermetic: every PR, pushes touching `tests/`, nightly, and each service repo's `service-image`
dispatch, which builds that service from its pushed branch) and the `quality-gate` job of
[`.github/workflows/_deploy-environment.yml`](../.github/workflows/_deploy-environment.yml)
(live `test` tenant, after every deploy of `tfvars_name=test-<env>`; see "Running against
the live `test` tenant" and [docs/RELEASE.md](../docs/RELEASE.md)).

## How authentication works in the suites

The platform has no test-mode auth. The suites use what production uses:

* **PATs** (`dvp_…`) are pure-Mongo credentials. `cmd/seed` writes three users
  and their keys straight into `users` / `api_keys` in the exact shape
  `services.APIKeyService` produces — the same approach as devplus-backend's own
  `cmd/ci-seed-pat`. Roles: `qa-admin` (tenant admin), `qa-user` (role=user, no
  membership), `qa-reader` (role=user with `read` on the seed project).
* **Cedar** runs for real: the true governance service with the backend's
  seeded system policies. The contract harness encodes the "User - No Role" and
  "User - Read Role" allow-lists (`contract/expectations.go`) and asserts every
  other operation is a 403 for those principals.
* **Browser login** goes through the real backend `/api/v1/auth/login` →
  WorkOS → `/callback` → sealed `devplus_session` cookie. Playwright only
  answers the hosted-login hop (the page a human would fill in); the backend's
  code exchange hits `mock-workos`, which impersonates `api.workos.com` via a
  compose network alias, TLS and `SSL_CERT_FILE`.

## Running against Tilt (local)

```bash
tilt up -f devplus-platform/Tiltfile -- --tenant=acme --mode=local   # as usual
cd devplus-platform/tests
export MONGODB_URI='mongodb://admin:admin123@localhost:27017/?authSource=admin' MONGODB_DATABASE=acme
go run ./cmd/seed                # writes tests/.seed.env (git-ignored)
go run ./cmd/smoke               # < 60 s
go test ./contract/... -v
go test ./regression/... -v      # journeys needing a mock (OIDC federation, executor dispatch, AI) skip on Tilt
cd e2e && bun install && bunx playwright test
cd ../perf && k6 run read-heavy.js
```

Every suite reads the same variables (defaults are the Tilt port-forwards):
`BACKEND_URL OIDC_URL OPENTOFU_URL AGENT_URL GOVERNANCE_URL FRONTEND_URL
MONGODB_URI MONGODB_DATABASE DEVPLUS_PAT DEVPLUS_USER_PAT DEVPLUS_READ_PAT`
plus the `SEED_*` / `PERF_*` ids `cmd/seed` emits. `DEVPLUS_REQUIRE_PLATFORM=1`
turns "platform unreachable" skips into failures (CI sets it);
`DEVPLUS_HERMETIC=1` does the same for the journeys that need a mock.

## Running against the live `test` tenant (TEST_MODE=live)

The `test` tenant is the long-lived canary in every environment
(`https://test.dev.devplus.io`, `https://test.stage.devplus.io`, `https://test.devplus.io`;
namespace and database `test`). It has real WorkOS AuthKit, MongoDB Atlas over OIDC
workload identity, Azure OpenAI in AI Foundry and Cloudflare ingress — nothing is mocked
and only the public ingress is reachable. The suites adapt automatically:

| | `TEST_MODE=mock` (compose / Tilt) | `TEST_MODE=live` (auto when `BACKEND_URL` is https) |
|---|---|---|
| Credentials | seeded into Mongo by `cmd/seed` | `DEVPLUS_PAT` (admin) from the environment; `DEVPLUS_USER_PAT` / `DEVPLUS_READ_PAT` optional |
| Fixtures | `cmd/seed -mode=mongo` | `cmd/seed -mode=api`: verifies/creates the project, generic connector, binding, a registry module (`terraform-aws-modules/s3-bucket/aws@4.1.2`, analysed by the real executor) and a catalog item, all through the REST API; writes `.seed.env` |
| Smoke | `-target=cluster`: every service port probed directly | `-target=edge`: `/api/v1/health`, `/healthz`, the SPA shell, the ingress-routed `/.well-known/*`; every route family through the API gateway; internal services **indirectly** — governance via a Cedar-gated read (and a member deny when `DEVPLUS_USER_PAT` is set), OIDC via the proxied JWKS, agent-service via the proxied thread-state route, opentofu-server + executor via a preview |
| Contract harness | full | full; the Cedar tier sweeps skip unless the low-privilege PATs are provided |
| Regression | every journey (72 test functions; the 2 live-only ones skip, the 5 `TestOutage*` ones need `DEVPLUS_CHAOS=1`) | Mongo-only assertions become optional deepenings, internal-service journeys (state lock, governance direct, OIDC token families, executor dispatch, agent-service direct) skip, and two live-only journeys run: the real executor picks up a deployment and completes a preview, and every internal service answers through the backend. The AI journey sends a safe read-only prompt to the real model and asserts the SSE contract (valid JSON frames with `type`, visible text, no error frame, no action card, stream completion) |
| Playwright | mock-workos login legs | strategy A (default): every `/api/v1` call the SPA makes is continued with `Authorization: Bearer $DEVPLUS_PAT` (the sealed session cookie cannot be minted from a PAT; WebSocket upgrades cannot carry the header, so the log terminal fills by REST polling). Strategy B: `LIVE_LOGIN=authkit` with `WORKOS_TEST_USER_EMAIL` / `WORKOS_TEST_USER_PASSWORD` types into the real AuthKit hosted form |
| k6 | 300 / 50+50 / 100+30 VUs | `LIVE_MODE=true`: 20 read VUs, 2×6 real deployments (the executor runs them), no lock scenario (opentofu-server is internal), 20 WebSocket + 2 SSE turns (under the per-user run cap of 3); p95 SLO 500 ms (`LIVE_P95_MS`), error rate still 0%; all writes inside `PERF_PROJECT_ID` |

```bash
export BACKEND_URL="https://test.dev.devplus.io"
export FRONTEND_URL="https://test.dev.devplus.io"
export DEVPLUS_PAT="<PAT_FROM_KEY_VAULT>"       # az keyvault secret show --vault-name kv-devplus-dev --name devplus-test-pat --query value -o tsv
export TEST_MODE="live"

cd devplus-platform/tests
go run ./cmd/seed -mode=api                        # verifies / creates the fixtures, writes .seed.env

# Run Smoke (<60s)
go run ./cmd/smoke -target=edge

# Run Contract & Regression
go test ./contract/... -v
go test ./regression/... -v

# Run Playwright E2E
cd e2e && bunx playwright test

# k6 (capped live profile)
cd ../perf && k6 run read-heavy.js && k6 run mutation-lock.js && k6 run streaming.js
cd .. && go run ./cmd/seed -purge-perf             # deletes the perf-*/qa-*/e2e-* deployments through the API
```

Or the same stages the pipeline runs: `tests/ci/run-suite.sh gate dev|stage|prod`.

**Bootstrapping a tenant's PAT (once per environment):** sign in to the tenant as
its first admin (`admin@devplus.io`, WorkOS AuthKit), open Settings → API keys →
Create key (365 days, not read-only), and store the token as the Key Vault secret
`devplus-test-pat` in that environment's vault — or as the GitHub Actions secret
`DEVPLUS_TEST_PAT_DEV` / `_STAGE` / `_PROD`. The gate looks at the GitHub secret
first and falls back to Key Vault. Rotate before expiry; the smoke run fails
with 401 when it lapses. For the Cedar tier sweeps, invite a second user
(role `user`), have them mint a PAT, and export it as `DEVPLUS_USER_PAT`.

What the live tenant accumulates: every deployment the suites create is run by
the real executor (the seed module has no cloud connector, so runs fail fast)
and deleted afterwards; `seed -purge-perf` cleans stragglers. Expect failed
jobs in the tenant's history — that is the executor being exercised.

## Running hermetically (Docker Compose)

From the umbrella directory (parent of `devplus-platform` and the sibling repos):

```bash
docker compose -f devplus-platform/tests/ci/docker-compose.yml up -d --build --wait
export MOCK_K8S_URL=http://localhost:6444 MOCK_AOAI_URL=http://localhost:9010 MOCK_WORKOS_URL=http://localhost:9020
devplus-platform/tests/ci/run-suite.sh         # or: run-suite.sh smoke|contract|regression|e2e|perf
```

Images build from the sibling checkouts with each Go service's `Dockerfile.tilt`
(no private-module token needed); agent-service and the frontend use their own `Dockerfile`. `PLATFORM_IMAGE_TAG=<version>` switches to
the published `ghcr.io/devplusinc/*` images (arm64 only).

### Mocks (`mocks/`)

| Mock | Stands in for | Protocol implemented | Control |
|---|---|---|---|
| `mock-workos` | WorkOS AuthKit / User Management | `GET /user_management/authorize` (302 back with `code=<email>`), `POST /user_management/authenticate` (code + refresh grants, RS256 access tokens, `iss=https://api.workos.com/user_management/<client_id>`), `GET /sso/jwks/<client_id>`, logout | TLS on the `api.workos.com` alias; CA written to the `certs` volume |
| `mock-aoai` | Azure OpenAI | `/openai/deployments/{d}/chat/completions` (streaming + non-streaming, `response_format.json_schema` and forced `tool_choice` answered with canned structured output per schema name: `PlanDecision`, `ArchitectMode`, `ArchitectureDesignDraft`, `RefinedModule`), `/embeddings` | `PUT /__mock/config {plan, architectMode, designCatalogItemId, reply, latencyMs}`, `GET /__mock/requests`, `POST /__mock/reset` |
| `mock-k8s` | Kubernetes API | `POST /apis/authentication.k8s.io/v1/tokenreviews` (static SA token map), `batch/v1` Jobs list/get/create/delete with label selectors and 409 on duplicates, Pod lookup for image discovery | TLS on :6443 for the services, plain :6444 for tests; writes `ca.crt`/`token`/`namespace` to the `k8s-sa` volume mounted at the in-cluster path |

Cloud provider and GitHub calls are not mocked: the operations that need them
are listed in `contract/expectations.go` (`externalDependencyOps`) and are
verified for mounting, authentication and authorization only.

## What each pillar asserts

**Smoke** — every service's health endpoint; the OpenAPI document parses; one
authenticated read per route family is not 5xx; every operation is mounted
(401 unauthenticated, or a non-404/405/5xx answer for public routes); a
temporary project is created, a throwaway state is LOCK/UNLOCK/DELETEd on
opentofu-server, OIDC JWKS has keys, governance evaluates, the project is deleted.

**Contract** — for every operation in `api/openapi.json`:
`TestNegativeAuthentication` (401 + ErrorResponse body), `TestCedarAuthorizationNoRoleUser`
(403 unless allow-listed; AI proxy + spec endpoint are auth-only),
`TestCedarAuthorizationReadMember` (read tier on the member project only),
`TestReadOnlyPATRejectsWrites`, `TestSecretRedactionInvariant` (planted AWS/Azure/generic
secrets never appear; `secretKey clientSecret token password rawKey secretHash privateKey
accessToken refreshToken apiKey` never populated on reads), `TestSchemaConformityReads`
and `TestSchemaConformityMutations` (status documented + body validates against the
3.1 schema via libopenapi-validator). `TestMain` fails the run if any operation was
touched by fewer than two checks. `robustness_test.go` adds four whole-surface sweeps:
truncated JSON to every body-taking operation (never 5xx, 400 carries the envelope),
a pattern-violating id in every path parameter (never 2xx/5xx),
CORS preflight per path (`Access-Control-Allow-Origin` + `authorization` allowed), and
spec drift: the deployed `/api/v1/openapi.json` must describe the same operations as
the harness copy, and the frontend's vendored `openapi/openapi.json` must match too.
`static_config_test.go` needs no running platform: it cross-checks the env names the
executor dispatcher injects into job pods against what the job code reads (and that the
`OPENTOFU_*` names the job reads are injected), and every `- name:` the Helm chart sets on a
service against that service's source.

**Regression** — `deployment_lifecycle_test.go` (project → connector → binding →
catalog validation report → template → preview row → submission → `deployment_queue`
document → reads/updates → drift/refine/adopt surfaces), `state_lock_test.go`
(A LOCK / B 423 with holder info / A POST ?ID / B POST 423 / A UNLOCK / B LOCK, a 12-way
LOCK race with exactly one winner, registry upload/versions/download/delete),
`ws_logs_test.go` (401 on unauthenticated upgrade, hello frame, `type:"log"` frames with
`deploymentId`, two-subscriber fan-out, read-member may stream but not broadcast),
`ai_sse_test.go` (SSE frames, `review_architecture_plan` action card with `designId`,
persisted design, thread-state pending action, resume with `{"approved": true}` →
`approved` + step states, the required audit record with outcome `approved` and the
Python-compatible approval hash, every agent-action outcome accepted, a `run_end` event
closing each turn),
`oidc_tokenreview_test.go` (401/403/200 paths of `/token/service`, JWKS signature,
claims, TTL, backend accepts the service JWT on `/api/v1/internal/*`),
`governance_reload_test.go` (deny-by-default, policy invisible until `POST /reload`,
loaded-policy listing, disable, and the same via the backend CRUD which triggers reload),
`executor_dispatch_test.go` (queue item leased by `dispatcher`, `executor-<id>` Job with
the executor env contract, label-selector back-pressure query, delete action),
plus `api_keys_test.go`, `identity_test.go`, `architecture_secrets_rag_test.go`, and the
second wave: `identity_security_test.go` (expired/revoked keys, deactivated owner → 403,
read-only + project-scoped key semantics), `modules_catalog_test.go` (git module registration
→ `module_queue` analyze item, deferred deletion, global/project catalog scoping, typed project
attributes and their 400 validation), `protocol_edges_test.go` (lock break via DELETE and
unlocked POST, WebSocket subscriber isolation, executor lease reclaim after a dead worker, OIDC
project/deployment token claims, governance evaluated with the backend's exact entity shape
across the admin/user/read/write/project-admin matrix), `ai_edges_test.go` (reject path → draft
+ cleared pending action + the `rejected` audit record, conversation delete, agent-service direct `/chat`, `/refine-module`
structured output, FastAPI 422/401 envelopes), and the third wave: `cookie_session_test.go`
(the SPA's real session: login 302 + state cookie, tampered state refused, callback seals
`devplus_session`, CSRF header required on writes, refresh, logout invalidates),
`internal_api_test.go` (`/api/v1/internal/*` accepts only `INTERNAL_API_TOKEN` or a
role=system JWT, returns plaintext secrets to the executor; a forged HS256 system token via the
`JWT_SECRET` fallback is pinned as a security note), `data_integrity_test.go` (project /
connector / catalog / module delete cascades and orphans pinned, catalog constraints shaping
deployments, duplicate names), `isolation_test.go` (read member cross-project 403 incl. WS
upgrade, WebSocket Origin policy, system-policy immutability (update, delete and toggle are
403), list validation: agent-actions limit, search clamp, invite validation, knowledge upsert),
`executor_kinds_test.go` (module-analyze and discovery Jobs with their env contract, resource
limits, queue lease) and `streaming_edges_test.go` (`: keepalive` reaches the client through
the backend proxy during a 24 s model stall; 2 MB state round-trips through envelope
encryption; re-upload replaces a registry tarball).

**E2E** — `e2e/tests/platform-journey.spec.ts` (login → catalog → deployment form → live log
terminal → AI action-card approval → logout), `access-control.spec.ts` (unauthenticated deep
link → `/api/v1/auth/login?return_to=…`, public signed-out page, admin navigation, a role=user
session kept out of `/admin/*` with the API refusing too) and `admin-journeys.spec.ts`
(settings API key create/show-once/use/revoke, admin project create/delete, generic connector
add/delete with secret never echoed, governance policy create → engine permits → disable →
engine denies → delete, teams create/add member/delete, invite issue + public token lookup,
deployments list Pending/filter/open, command palette + project selector, AI plan rejection →
design back to draft), `product-surfaces.spec.ts` (CLI sign-in hand-off `/cli-auth` → scoped
key + loopback `/done?code=` capture + public redeem, malformed link refused; admin registry
import → detail → delete version; catalog template authoring visible to deployers; deployment
rename/delete; architecture principle create/toggle/delete; profile name round-trip; project
settings description + tags; resources/secrets screens; 404 page; theme toggle persisted) and
`accessibility.spec.ts` (axe-core WCAG 2.x A/AA over ten authenticated screens; serious and
critical violations fail unless pinned in `KNOWN_A11Y` with a per-rule node cap; none is pinned
since devplus-frontend#114 was fixed). Every UI step is cross-checked through the API. The specs follow the redesigned shell's
conventions (design system, September 2026): page CTAs read "New <thing>" (`New API key`,
`New project`, `New policy`, `New template`, `Import module`), row actions live in a
`getByRole("button", { name: "Actions for <name>" })` menu (`More actions for <name>` on
detail pages), destructive steps confirm in an `alertdialog` whose button repeats the verb
(`Delete project`, `Delete version`; deployment delete is type-to-confirm), success is a sonner
toast (`Template created`, `Invite created`, `Module "…" imported`), the deployment form is a
two-step wizard (`Continue` → `Create deployment`), the log terminal is
`getByRole("log", { name: /Deployment logs/ })`, the account menu is
`Account menu for <name>` in the sidebar footer, the project switcher is
`Switch project. Current: <name>`, and the theme control is `Theme: <mode>` with
`menuitemradio` items. Stable ids that survived: `#param-<name>`, `#deployment-name`,
`#rename-deployment-name`, `#name`, `#sr-name`, `#github-url`, `#module-name`,
`#template-module`, `#template-version`. `fixtures/platform.ts` exports `rowAction`,
`confirmDialog`, `expectToast` and `openAccountMenu` for these patterns. The SPA ships almost
no `data-testid`s; adding them to these elements is the recommended hardening.

**Performance** — thresholds live at the top of each script; results are written
to `perf/perf-results/*.json`. `mixed-workload.js` holds a ramping *arrival rate* (not a VU
count) over an 80/15/5 read/detail/mutation mix so a slower backend cannot hide behind fewer
iterations; live mode caps it at `LIVE_MIXED_RPS` (10 req/s). `streaming.js` counts fan-out loss (every subscriber must
receive every broadcast line) and measures SSE time-to-first-token.

## Known platform behaviours pinned by the suite

Deliberate behaviours the tests encode as expectations:

* `POST /api/v1/deployments` (global create) always answers 500 once validation passes
  (documented as a non-functional legacy route in devplus-backend `internal/server/routes.go`);
  pinned in `regression/deployment_lifecycle_test.go`.
* `POST /api/v1/groups` returns 200, not 201; `POST /api/v1/rag/knowledge` is an upsert (200).
* `PUT /api/v1/governance/policies/{id}` is a partial content update; `enabled` is owned by
  `/toggle` (`{"enabled": bool}`).
* opentofu-server has no lock TTL: `DELETE /state/{p}/{n}` is the only way to break a lost lock.
* governance `POST /reload` returns 200 even if Mongo is unreachable (and then serves zero policies).
* The invite token is returned on `GET /api/v1/invites` (it is the invitation link).
* OpenBao is no longer deployed by the Helm chart (envelope encryption replaced it), so the
  smoke suite does not probe port 8200.
* Deleting a `pending` deployment (never provisioned) keeps the row in `deleting` and enqueues
  a `delete` job for the executor. In the hermetic stack no job pod runs, so those rows and
  queue items accumulate; the k6 mutation suite produces thousands. `go run ./cmd/seed -purge-perf`
  removes the `perf-*` deployments and their queue items (`ci/run-suite.sh` calls it after the
  perf stage; the executor journey calls the same purge before asserting on its own lease).
* mock-k8s persists its CA and server key pair in the `k8s-sa` volume so a restart of the mock
  keeps the trust chain that oidc-service and the executor loaded at startup. If you rebuild the
  volume, recreate those two services as well.
* mock-workos mints a fresh CA + RSA signing key on each start and writes the CA to the shared
  `certs` volume. The backend reads that CA (SSL_CERT_FILE) and caches the JWKS at startup, so if
  mock-workos restarts on its own (crash/OOM + restart policy) while the backend keeps running, the
  login flow breaks with `/callback -> 401 "Authentication failed"` (stale CA/JWKS). Fix: restart
  the backend (and oidc-service, executor) to re-read the current CA. Symptom test:
  `regression/cookie_session_test.go`.

## Defects found by the suite

A defect the suite finds is pinned (`contract/expectations.go` → `knownDefects`,
`knownUnvalidatedPathParams`, or a `known defect (pinned)` branch in a regression test) so the
suite stays green, and the pin FAILS when the behaviour changes: the fix removes the pin and
turns it into an assertion of the right behaviour. Nothing is pinned in `knownDefects` or
`knownUnvalidatedPathParams` today; the round 4 findings (#18 to #24 below) are pinned in the
regression and contract tests. Filed as GitHub
issues on 2026-09-14: devplus-backend #50 (1–4), #51 (5), #52 (6), #54 + devplus-agent-service
#34 (7), #53 (8), devplus-frontend #113 (9), devplus-backend #55 (10); round 3
(2026-09-15): devplus-backend #56 (11), #57 (12), #58 (13), #59 (14),
devplus-executor #56 (15), devplus-platform #66 (16), devplus-frontend #114 (17).

| # | Where | Was | Now |
|---|---|---|---|
| 1 | `PUT /api/v1/projects/{projectId}` on an unknown id | 500 "Failed to update project" | 404 (devplus-backend #68, 0.1.16) |
| 2 | `PUT /api/v1/groups/{groupId}` on an unknown id | 500 "Failed to update group" | 404 (devplus-backend #68, 0.1.16) |
| 3 | `PUT /api/v1/connectors/providers/generic/{id}` on an unknown id | 500 | 404 (devplus-backend #68, 0.1.16) |
| 4 | `PUT`/`DELETE`/`PUT …/toggle` on `/api/v1/governance/policies/{id}` for an unknown id | 500 | 404 (devplus-backend #68, 0.1.16) |
| 5 | `DELETE /api/v1/auth/photo` | 500 "Failed to delete photo", even right after a successful upload | 200 (devplus-backend #69, 0.1.16) |
| 6 | `POST /api/v1/groups` with `{}` | 200 and an empty-named group; a duplicate name was a 500 | 400 "Name is required" / 409 (devplus-backend #68, 0.1.16) |
| 7 | Agent approval audit trail | agent-service recorded `proposed`/`approved`/`rejected`; the backend accepted only `succeeded`/`failed` and answered 400, which the agent swallowed: no human-in-the-loop decision was audited | the shared vocabulary on both sides; the journeys require the `approved` and `rejected` records (devplus-shared #47, devplus-backend #67, devplus-agent-service #40, 0.1.16) |
| 8 | Backend → governance under load | `services/governance_client.go` wrapped `http.DefaultTransport` (2 idle conns/host); at ~300 req/s every Cedar check opened a new TCP connection and the backend ran out of ephemeral ports (`connect: cannot assign requested address` → 500 "Authorization service error") | one pooled transport for every outbound client, and replies read to the end so connections are reused (devplus-backend #66, 0.1.16) |
| 9 | `devplus-frontend` `main` @ `046313e` (2026-08-31 dependency bump) | `bun run build` failed on a `@tanstack/ai-react` type change, so no SPA image built | builds since the September redesign |
| 10 | `POST /api/v1/projects` with an invalid attribute key/type | 500 "Failed to create project" | 400 "Invalid project attributes" (devplus-backend #68, 0.1.16) |
| 11 | `GET …/{projectId}/catalog`, `GET …/{projectId}/modules`, `POST …/deployments/{id}/logs/broadcast` | a garbage id (`not-a-valid-id!`) was accepted with 200 | 404 at the router: ObjectID path parameters carry the 24-hex pattern (devplus-backend #69, 0.1.16) |
| 12 | `PUT /api/v1/governance/policies/{id}/toggle` on a system policy | no `isSystem` guard: the admin could disable "Global Admin Full Access" and was then locked out of re-enabling it; `DELETE` on a system policy was a 500 | 403 "Cannot modify system policy" for update, delete and toggle (devplus-backend #68, 0.1.16) |
| 13 | `POST …/deployments` for a catalog item with `requiredConnectorTypes` and no `connectorId` | 500; the "requires a connector" reason was only logged | 400 naming the requirement (devplus-backend #68, 0.1.16) |
| 14 | `POST /api/v1/invites` | accepted `not-an-email`, `Name <x@y>` and `role: superuser` | 400 (devplus-backend #69, 0.1.16) |
| 15 | executor `services/job_dispatcher.go` | injected `TERRAFORM_WORK_DIR` / `TERRAFORM_REGISTRY_URL`; the job reads `OPENTOFU_*`, so both fell back to code defaults | injects the `OPENTOFU_*` names (devplus-executor #62, 0.1.16) |
| 16 | Helm `backend.yaml` | set `GITHUB_CALLBACK_URL` (a stale path) that the backend never reads | dropped (devplus-platform #69, 0.1.15) |
| 17 | SPA accessibility | `ProjectSelector` combobox had no accessible name, no skip link, secrets `<svg role=img>` without alt, admin users tabs `aria-controls` dangling | fixed by the redesign; the last pin went in 0.1.16 when a run against the release showed it gone (devplus-frontend #123 holds it in the SPA's own tests) |

Security notes pinned without an issue: an HS256 token signed with `JWT_SECRET` is accepted as a
system credential on `/api/v1/internal/*` (`internal_api_test.go`), so that secret must be strong
and rotated; the Tiltfile default must never reach a real environment.

| 18 | `GET /api/v1/projects/{projectId}/deployments/{id}` | ignores `{projectId}` and fetches by `{id}` alone, so a user with read on **any** project reads **every** deployment via their project's path (IDOR) — the delete path resolves the real project and is correctly refused | 404 when the deployment's project != the path project |
| 19 | `POST /token/project/{id}` and `/token/deployment/{id}` (devplus-oidc-service) | **unauthenticated** and merge caller-supplied `claims`, so anyone who can reach the service mints a `role=system` token the backend honours on the internal secret surface; only network isolation guards it | require the backend secret like `/token/service`; do not merge privilege claims |
| 20 | module `gitUrl` on `POST /api/v1/modules` | accepts `file://`, link-local/metadata `http://`, and `ssh://` with no allowlist; the executor `git clone`s it (SSRF / local-file read) | scheme/host allowlist; `--` before the URL in the clone |
| 21 | `POST /api/v1/projects` | `name` and `description` have no length bound; a 2 MB name persists (storage/DoS, UI degradation) | bound the fields (compare the support desk's 200 / 20000 limits) |
| 22 | cost-history (`GET …/deployments/cost-history`) | a populated `cost` serializes `projects:null` while the OpenAPI schema declares a non-nullable array, so the response violates its own contract | `omitempty` / empty slice / nullable in the spec |

Round 4 issues (2026-10-03, backend at 200 operations, all repos `main`):
devplus-backend #72 (18), devplus-oidc-service #35 (19), devplus-backend #73 (20), #74 (21), #75 (22).
#18–#21 are security findings (two confirmed cross-boundary: an IDOR and an unauthenticated
privilege-escalation mint). #22 is exempted in the schema-conformity sweep
(`contract/expectations.go` → `knownSchemaDefects`) and pinned by the feature test.

| 23 | deployment by-id handlers (`Update`, `Delete`, `DriftCheck`, `Refine`, `BroadcastLog`, `logs/stream`) | authorize Cedar against the path `{projectId}` but resolve the deployment by `{id}` alone — a **project-admin of any project** can update/destroy/re-provision/broadcast-into/subscribe-to **any** other project's deployment via their own project's path (critical BOLA; destructive). `Delete` was initially mis-read as safe — the 403 seen at the write tier is the admin-tier gate, not an ownership check | load by id, then 404 unless `deployment.ProjectID == {projectId}` |
| 24 | `GET /api/v1/projects/{projectId}/catalog` | scopes by the `?projectId` query (and lists **all tenant** items when it is absent), not the path — a reader of one project reads every project's catalog items (modules, params, setup scripts) | scope strictly by the path project |

Round 4 security additions (2026-10-03): devplus-backend #76 (23, the critical write/delete BOLA family; #72 is its read sibling),
#77 (24). The full BOLA family is pinned
in `regression/authorization_scoping_test.go` (`TestDeployment{Read,Write,Delete}IsScopedToItsOwnProject`,
`TestCatalogListIsScopedToPathProject`). Checked and found NOT vulnerable: cross-project delete at the
*write* tier (it needs admin), CORS reflection, and the AI assistant proxy (auth-only by design, but
agent-service forwards the caller's token so downstream data access stays Cedar-scoped). The WebSocket
log-stream leg is exercised live (`TestDeploymentLogStreamIsScopedToItsOwnProject`), and
`TestNonAdminCannotEscalatePrivilege` locks in the privilege boundary (a role=user principal
is refused on role self-update, invites, user/group/policy creation and admin api-keys — all
verified denied, no bug). Also checked clean: pagination cursor tampering, read-only key
enforcement across all 200 ops, and role self-escalation.

`perf/read-heavy.js` keeps its `0%` error threshold on purpose: it failed until #8 was fixed (0.1.16).

### Round 4 additions (2026-10-03)

New regression journeys: `concurrency_test.go` (parallel creates / deletes / revokes /
toggles / renames / member-adds — no 5xx, no duplicates, convergent final state),
`resilience_test.go` (opt-in `DEVPLUS_CHAOS=1`: stop one dependency at a time — governance
fails closed, agent-service degrades AI only, opentofu-server does not block submission,
Mongo restart recovers, a Kubernetes outage keeps queued work — then recovers),
`authorization_scoping_test.go` (a deployment's own project is the authority, not the path:
delete is correctly scoped, read is the pinned IDOR #72), `features_test.go` (cost history incl.
a seeded cost event, GCP connector create-validation and key redaction, repository-analysis read
scoping, the support desk's not-configured contract), and `internal_api_test.go`'s
`TestUnauthenticatedOIDCMintGrantsSystemAccess` (#35). New contract file `security_edges_test.go`
(module gitUrl schemes #73, CORS is not reflected, oversized fields #74, the unauthenticated
opentofu-server boundary). `robustness_test.go` gained two whole-surface guarantees:
`TestReadOnlyPATRejectsEveryWrite` (an admin-minted read-only key is refused on every mutating
operation — proving the read-only flag is enforced platform-wide, independent of Cedar) and
`TestListCursorTamperingIsRejected` (every `cursor` list answers 4xx/empty, never 5xx, to a
forged cursor). Both passed across all 200 operations — no new defect, debt locked out. New browser spec `new-surfaces.spec.ts` (support not-configured, GCP
connector form, dashboard time range, `?`/`g` keyboard navigation, the force-delete option, the
phone sidebar sheet).

## Verified on the hermetic stack (2026-09-27, all repos at `main`, Apple Silicon, Docker Compose)

Re-verified after pulling every sibling repo to `main` (backend 184 operations incl. GCP
connectors, HubSpot support desk and cost history; the fully redesigned SPA; the
repo-analysis-v2 shared models; force delete across executor/agent/frontend).

| Stage | Result |
|---|---|
| `cmd/smoke` | 227 probes (25 route families incl. Support), 0 failed, ~0.4 s |
| `contract/` | 18 tests / 948 subtests pass (6 skipped: GitHub App and HubSpot not configured); coverage ledger complete for all 184 operations; GCP service-account key planted and never echoed; 15 pinned findings still fired (fixed in 0.1.16, see above) |
| `regression/` | 48 journeys pass (3 live-only skipped), 76 subtests; the "generic" connector requirement is now a no-op upstream, so the requirement test gates on `aws` |
| `e2e/` | 38 browser tests across 5 spec files rewritten for the redesigned SPA: 35 pass, 3 live-only skipped; axe gate down to one pinned rule (admin users tabs `aria-controls`) — the skip link, single `<main>`, named switcher and labelled selects all landed upstream |
| `perf/read-heavy.js` (10 VUs, 15 s) / `perf/mixed-workload.js` (40 req/s, 25 s) | 0 failed requests, all thresholds met |
| `perf/mutation-lock.js`, `perf/streaming.js` | unchanged routes; last full run 2026-09-15 |
| `perf/read-heavy.js` at 300 VUs | failed on defect #8 (backend ↔ governance connection churn), fixed in 0.1.16 |

Live mode was verified by simulation against the same compose stack with `TEST_MODE=live`
(ingress-less, so no real Cloudflare/WorkOS/Azure OpenAI in the loop): `seed -mode=api`
resolves the identical fixture ids through the API; `smoke -target=edge` passes every probe
except the three that need the real ingress or nginx (`/.well-known/*`, `/healthz`); the
contract harness completes its ledger; the regression suite passes with every internal-only
journey skipping by design and the live AI conversation journey running through the backend;
the k6 live profile caps to 20 / 2×6 / 20+3 VUs with all thresholds met; Playwright runs in
the PAT strategy. Journeys that need a real executor (`TestLiveExecutorPicksUpDeployment`,
the executor-fed log terminal) are only exercised on a real tenant. The round-3 additions
behave the same way in the simulation: the robustness, spec-drift and static-config contract
sweeps pass unchanged; the regression journeys that inspect internal ports, Mongo, the
session cookie flow or the internal API skip by design, and `TestWebSocketOriginPolicy`
skips over plain http (only a real tenant sets `TENANT_HOSTNAME`, where a foreign Origin
must be a 403 — a security failure otherwise).
