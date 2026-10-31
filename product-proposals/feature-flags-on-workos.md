---
id: 73919343-8b04-4412-a2bb-c6bf864475d0
binder: binder-mvvle32a-1
folder: folder-mvw0u8o0
---
Feature Flags on WorkOS

**Target Release:** unscheduled (potential feature)
**Status:** Proposed
**Scope:** `devplus-shared`, `devplus-backend`, `devplus-frontend`, `devplus-agent-service`, `devplus-executor`, `devplus-mcp`, `devplus-cli`, `devplus-sdk-go`, `devplus-platform` (+ optional `devplus-governance-service`)

---

## Overview & Problem Statement

DevPlus turns features on and off five ways today: Helm values rendered to env vars, Terraform booleans, env vars read at boot, the per-tenant `tenant_settings` document, and per-entity `enabled` fields. All of them need a redeploy or a tenant admin. None of them can say "turn the Architect on for these three tenants" or "kill repo analysis everywhere, now". This feature adds one mechanism for that, backed by **WorkOS Feature Flags**, without adding a service or a vendor.

## 1. Why WorkOS fits

**tenant = namespace = Mongo DB = WorkOS organization.** WorkOS flags target organizations and users. So a flag targeted at an org is a per-tenant toggle, with no mapping table. Every per-developer tenant (`saamer`, `aaron`, `helder`, `test-*`) is its own org, which lets us roll out per developer, then dev, stage and prod.

What WorkOS gives us (checked against the docs on 2026-10-25):

| Capability | Detail |
|---|---|
| Flag object | `slug`, `name`, `description`, `tags[]`, `enabled`, `default_value`. Flags are created in the dashboard for every environment of the project at once. |
| Targeting | Per environment: rule None / Some (pick orgs and users) / All. A flag switched off returns false everywhere and keeps its targeting, so the toggle itself is a kill switch. |
| Access token claim | `feature_flags`: the slugs enabled for the user, from org and user targeting. Refreshed only at sign-in or session refresh. The JWT template budget is 3072 bytes, shared with roles and permissions. |
| `GET /organizations/{orgId}/feature-flags` | Flags enabled for an org. Paged (`limit` 1-100, `before`/`after`). |
| `GET /user_management/users/{userId}/feature-flags` | Flags enabled for a user, directly or through their orgs. |
| `GET /feature-flags`, `GET /feature-flags/{slug}` | List or read flag definitions. |
| `PUT /feature-flags/{slug}/enable` / `disable` | Switch a flag per environment. |
| `POST` / `DELETE /feature-flags/{slug}/targets/{resourceId}` | Add or remove an org (`org_…`) or user (`user_…`) target. |
| Runtime client | Node only (`@workos-inc/node` ≥ 12): in-memory, polls every 30 s. Custom target types (such as `project`) are evaluated only by this client. |

Constraints this leaves us with:

1. **No Go runtime client, and our Go SDK can't read flags.** The backend pins `workos-go/v4 v4.46.1`, which has no flags API. Flag support starts in `workos-go/v10`. Upgrading six major versions of the auth-critical SDK is its own project, so we call the two REST endpoints directly (§4.2).
2. **The claim is stale until the session refreshes,** and PATs (`dvp_`), agent run tokens (`dpr_`), internal OIDC callers and job pods don't carry it at all. So the backend reads the org's flags from the API. It uses the claim only to add user-targeted flags for session callers.
3. **Custom targets (per project or per deployment) need the Node client.** We leave them out. Flags apply per tenant and per user. Anything finer stays a tenant setting or a field on the entity.
4. **Flag definitions are created in the dashboard.** The API reads, enables/disables and targets flags but doesn't create them. A CI check (§8.2) keeps the code registry and WorkOS in step.

## 2. Principles

1. **The backend is the only evaluator.** It already owns authN, tenancy and the route table. Frontend, agent-service, MCP, CLI and executor read flags from it. None of them gets WorkOS credentials.
2. **When a flag is missing, behavior is normal.** Release flags are off until targeted. Kill switches are separate `kill-*` flags that are off until targeted. So a WorkOS outage, a typo'd slug or an empty list falls back to "nothing new, nothing killed", never "everything off". (§3.2)
3. **Flags can only narrow security, never widen it.** Fail modes (`PLAN_GATE_FAIL_MODE`, `UNREADABLE_FORBID_FAIL_CLOSED`), `controlPlane`, NetworkPolicies, auth issuers and approval rules stay in code, Helm and Terraform. A flag may switch a feature off, or switch on a feature that still passes Cedar. It may never skip a gate.
4. **The platform env still wins.** An env kill switch (`REPO_ANALYSIS_ENABLED=false`) beats any flag. Order: platform env off → WorkOS → code default. Every result reports which layer decided it, as `ResolveSchedule` already does.
5. **Flags are for DevPlus operators, tenant settings are for tenant admins.** A flag decides whether a tenant has spend sync. `tenant_settings.costSync` still decides whether that tenant's admin turned it on. The two compose and neither replaces the other.
6. **The backend enforces, other clients only display.** A hidden nav item is UX. The authoritative check is the route guard in the backend (§4.4).

## 3. The flag registry

### 3.1 Where it lives

New package `devplus-shared/featureflags`, tagged in a devplus-shared release. It holds the catalog of every flag the code knows, so Go services, codegen and CI all share one list:

```go
// Package featureflags is the catalog of DevPlus feature flags and the
// resolver that evaluates them. WorkOS holds the targeting; this package
// holds what each slug means and what it falls back to.
package featureflags

// Kind says how a flag behaves when nothing targets it.
type Kind string

const (
	// KindRelease gates new work: off until a tenant or user is targeted.
	KindRelease Kind = "release"
	// KindEntitlement gates a product surface a tenant is given.
	KindEntitlement Kind = "entitlement"
	// KindKill switches an existing feature off: targeting it kills the feature.
	KindKill Kind = "kill"
)

type Flag struct {
	Slug        string
	Kind        Kind
	Description string
	// Fallback is used only while WorkOS has never answered for this
	// tenant (no live poll, no stored snapshot).
	Fallback bool
	// PlatformEnv, when set, is an env var whose explicit "false" turns
	// the feature off whatever WorkOS says (principle 4).
	PlatformEnv string
	// Surfaces lists who reads the flag: backend, frontend, agent, mcp, executor.
	Surfaces []string
	Owner    string
	// RemoveBy is the date a release flag should be gone by.
	RemoveBy string
}

var (
	AIArchitect = register(Flag{Slug: "ai-architect", Kind: KindEntitlement, ...})
	KillRepoAnalysis = register(Flag{Slug: "kill-repo-analysis", Kind: KindKill,
		PlatformEnv: "REPO_ANALYSIS_ENABLED", ...})
)
```

A `go generate` step writes the same catalog to `featureflags.json`. The backend embeds that file in the OpenAPI document as the `FeatureSlug` enum, so the generated TypeScript client and SDK get typed slugs with nothing typed in by hand.

### 3.2 Naming and semantics

- Slugs are lowercase kebab-case (WorkOS's rule), prefixed by area: `ai-`, `deploy-`, `discovery-`, `cost-`, `health-`, `admin-`, `kill-`.
- Release and entitlement flags are **positive** (`ai-architect`): enabled means the feature is there.
- Kill switches are **separate and positive** (`kill-repo-analysis`): enabled means the feature is killed. We don't express a kill switch as removing a target from a default-on flag, for two reasons. The org list endpoint returns only flags *enabled* for the org, so "absent" has to mean "normal". And whether default-on flags show up in that list isn't documented clearly enough to rely on.
- WorkOS tags carry `kind:<kind>` and `owner:<team>` so the dashboard can be filtered.

### 3.3 Initial catalog

Taken from the toggles in use today (file references are in the appendix):

| Slug | Kind | Replaces or wraps | Surfaces |
|---|---|---|---|
| `ai-assistant` | entitlement | nothing yet: /ai, the assistant proxy and the agent service are always on | backend, frontend, agent, mcp |
| `ai-architect` | entitlement | always built in `graph.py` | backend, frontend, agent |
| `ai-policy-agent` | entitlement | always built | agent, frontend |
| `ai-principles-agent` | entitlement | always built | agent, frontend |
| `repo-analysis` | entitlement | `REPO_ANALYSIS_ENABLED` stays as the platform layer. The flag adds per-tenant control and hides the nav item, which today 503s instead. | backend, frontend, executor |
| `cost-spend-sync` | entitlement | `COST_SYNC_ENABLED` stays as the platform layer. The flag decides availability; `tenant_settings.costSync` keeps the admin's choice. | backend, executor, frontend |
| `support-desk` | entitlement | presence of `HUBSPOT_ACCESS_TOKEN` (still required) | backend, frontend |
| `log-streaming` | entitlement | always on | backend, frontend |
| `mcp-access` | entitlement | `AUTHKIT_ISSUER` still decides whether the backend accepts AuthKit tokens. The flag decides whether this tenant's MCP calls are served. | backend, mcp |
| `discovery-github-enrich` | release | `GITHUB_INVENTORY_ENRICH` | executor |
| `health-aws-health-api` | entitlement | `HEALTH_AWS_HEALTH_API` | executor |
| `kill-drift-checks` | kill | `DRIFT_CHECK_ENABLED=false` stays as the platform kill | backend |
| `kill-health-scans` | kill | `HEALTH_SCHEDULE_ENABLED` | backend |
| `kill-module-refresh` | kill | `MODULE_REFRESH_INTERVAL_MINUTES=0` | backend |
| `kill-ai-writes` | kill | new: the agent can read but every write tool is refused | agent, mcp |
| `kill-deploy-apply` | kill | new: plans still run, applies are held with a "paused by DevPlus" reason (a pause, so it narrows security) | executor |

Each new feature from now on starts with a `release` flag, and the flag is removed once the feature reaches All in prod (§8.4).

**Never flags:** `otel.enabled`, `mongodb.*`, `controlPlane.enabled`, `networkPolicy.*`, `serviceAccount.*`, `PLAN_GATE_FAIL_MODE`, `UNREADABLE_FORBID_FAIL_CLOSED`, `MCP_REQUIRE_AUDIENCE`, OIDC allow-lists, schedule intervals and bounds. These are infrastructure or security posture and stay in Helm and Terraform under review.

## 4. Backend: evaluation and enforcement

### 4.1 Resolution

```
                 ┌─ PlatformEnv explicitly "false" ──────────────► off   (source: platform)
evaluate(slug) ──┤
                 ├─ WorkOS answered (live or snapshot) ──────────► slug ∈ orgFlags ∪ userFlags
                 │                                                       (source: workos-org | workos-user)
                 └─ never answered ──────────────────────────────► Flag.Fallback  (source: default)
```

For a kill flag, the feature's effective state is the opposite of the flag. The resolver returns both values so the API stays unambiguous:

```go
type Effective struct {
	Slug    string `json:"slug"`
	Enabled bool   `json:"enabled"` // the flag's own value
	Active  bool   `json:"active"`  // is the feature usable (inverted for kill flags)
	Source  string `json:"source"`  // platform | workos-org | workos-user | default
	AsOf    time.Time `json:"asOf"`
}
```

This is `ResolveSchedule` (`devplus-backend/services/schedule_policy.go:181`) reduced to booleans: layers from most specific to least, a platform override on top, and the source attributed. The `models.EffectiveSchedule` source-attribution pattern carries over as is.

### 4.2 `FlagService` (devplus-backend/services/feature_flag_service.go)

- **Org flags.** One goroutine per replica polls `GET /organizations/$WORKOS_ORG_ID/feature-flags?limit=100` (following `after` cursors) every `FEATURE_FLAGS_POLL_SECONDS` (default 30, min 10). The set is held in an `atomic.Pointer`, so lookups cost nothing. Load: tenants × replicas × 2 requests/min, which is trivial.
- **Last-known-good snapshot.** Each successful poll that changes the set writes it to `tenant_settings.featureFlags` (`{slugs[], fetchedAt, etag}`), a new sub-document in the existing per-area `$set` style (`devplus-shared/models/schedules.go:27`). At boot the snapshot is loaded before the first poll, so a pod that restarts during a WorkOS outage keeps the tenant's flags. A failed poll keeps the current set, logs at warn, backs off exponentially, and bumps an OTEL counter `devplus.feature_flags.poll_errors`.
- **User flags.**
  - Session and AuthKit callers: read `feature_flags` from the verified token. Add `FeatureFlags []string` (`json:"feature_flags"`) to `WorkOSClaims` (`handlers/workos_jwks.go:32`) and put it in context (`contextkeys.FeatureFlags`) in `serveWorkOSIdentity`.
  - PAT and run-token callers: look up the user's `workosId` through `GET /user_management/users/{id}/feature-flags`, cached for 60 s per user (LRU, singleflight, like the refresh path in `auth_session_handler.go:82`).
  - Internal and system callers: org flags only.
- **Change audit.** When a poll differs from the previous set, write one `audit_events` record per changed slug (`feature_flag.enabled` / `feature_flag.disabled`, actor `workos`). This puts flag history next to the security audit log the tenant admin already reads. WorkOS's own dashboard history and Slack notifications cover who flipped it.
- **Transport.** A small `devplus-shared/featureflags/workos` client (two GETs, `Authorization: Bearer $WORKOS_API_KEY`, base URL from `WORKOS_API_BASE_URL` so the test mock can stand in). Swap it for `workos-go/v10`'s `FeatureFlags()` when the SDK upgrade lands; the interface is the same.
- **Local mode** (Tilt, no WorkOS): `FEATURE_FLAGS_OVERRIDES="ai-architect,kill-drift-checks"` replaces the WorkOS layer completely, and the source reads `override`. The chart only renders it when `featureFlags.overrides` is set, and refuses to render it when `global.environment` is `prod`.

### 4.3 Exposure

1. **`GET /api/v1/auth/me` gains `features`.** The response becomes `{...User, features: {"ai-architect": true, "repo-analysis": false, ...}}`, holding the `Active` values for every flag with the frontend, agent or mcp surface. The user fields stay where they are, so existing callers (frontend, agent-service `_require_me`, CLI) keep working. The agent service gets flags with no new call.
2. **`GET /api/v1/features`** returns the full `[]Effective` with source and `asOf`, for admin and debug use (the CLI's `devplus features`, and the Admin › Settings "Platform features" read-only panel). Readable by any authenticated user. Its opdef action is `models.ActionReadFeatures`.
3. **Cedar context (phase 2).** Add `features: Set<String>` (active slugs) to `cedarBaseContext` (`handlers/cedar_context.go:20`), declare it in `requestContextAttrs` and `planGateContextAttrs` (`services/governance_schema.go:52-85`), and add it to `loggedContextKeys` (`devplus-governance-service/services/governance_eval.go:133`). Tenant admins could then write `forbid … when { !context.features.contains("ai-architect") }`, and decision-log simulation would keep it. Optional. Route enforcement (§4.4) doesn't depend on it.

### 4.4 Enforcement in the route table

`api.Op` already has a `Flags api.OpFlags` field (`Public`, `NoOptions`), so the new field is called **`Feature`**, to avoid two meanings of "flag":

```go
reg.Op(api.Op{
	Method: http.MethodPost, Path: "/repositories/{repoId}/analysis",
	Action:  models.ActionStartRepoAnalysis,
	Feature: featureflags.RepoAnalysis.Slug,
	...
})
```

- The registry wraps the handler in `RequireFeature(slug)`. It runs after authN, so user flags are known, and before Cedar. An inactive feature answers **404** with `{"error":"feature_unavailable","feature":"repo-analysis"}`: the feature doesn't exist for this tenant, and the 503s it replaces read as outages. Add 404 to the op's `Errors` automatically, and document the body in `ErrorBodies`.
- A contract test walks the registry, like the `RequireProjectResource` one. Every `Feature` must be a registered slug that includes the backend surface. The OpenAPI operation gets `x-devplus-feature: <slug>` so SDK, CLI and MCP can explain a 404.
- Section-level gating for whole areas: `aiSec.RequireFeature(featureflags.AIAssistant)` on the assistant proxy section (`routes.go:197`).
- Schedulers (drift, health, module refresh, spend sync) run in the backend. Each sweep asks `FlagService` before it enqueues, so a `kill-*` flag stops new work within one poll. Work already queued finishes; §6 explains why.

## 5. Consumers

### 5.1 Frontend (devplus-frontend)

The bundle is static and reads no env (`Dockerfile:27-45`), so flags come from `/auth/me`.

- `contexts/UserContext.tsx`: keep `features` from `getCurrentUser()` and expose `useFeature(slug: FeatureSlug): boolean`. The `FeatureSlug` type comes from the generated `lib/api/schema.d.ts` enum, so a typo fails `tsc`. Refetch on window focus and every 5 min, so a flip shows up without a reload.
- `components/layout/navigation.ts`: `NavItem` gains `feature?: FeatureSlug`, and `useAppNavigation()` drops items whose feature is inactive. This works the way `supportConfigured` gates Help today (`navigation.ts:82`). Support becomes `feature: "support-desk"` alongside the configured probe.
- `src/routes.tsx`: a `<FeatureRoute feature="repo-analysis">` wrapper renders a "Not available on this workspace" page instead of the lazy page. It's the same shape as `AdminRouteGuard`.
- Error handling: `lib/http.ts` maps `404 feature_unavailable` to a typed `FeatureUnavailableError`, so cards that today treat a 503 as unavailable (`CostDriversCard.tsx:47`, `ModuleCandidatesPanel.tsx:94`) share one branch.
- Admin › Settings gets a read-only "Platform features" panel from `GET /features` that shows the source. A tenant admin can see that something was turned off by DevPlus, not by them.

### 5.2 Agent service (devplus-agent-service)

- `_require_me` / `_resolve_me` (`api/routers/chat.py:126`, `api/utils/stream.py:743`) already fetch `/auth/me` per run. Carry `features` into `AgentState`.
- `build_graph` (`domain/graph/graph.py:191`): build only the agents whose flag is active (`architect` → `ai-architect`, `policy` → `ai-policy-agent`, `principles` → `ai-principles-agent`; the rest are core). Pass the same set to the planner prompt, so the router never picks an agent that isn't there. `AGENT_NODES` becomes the full set, and the edge wiring (`graph.py:300-304`) iterates the built subset.
- `kill-ai-writes`: the tool wrapper refuses every write tool with a clear "paused by DevPlus" message, before the interrupt card. Backend Cedar stays authoritative either way.
- No WorkOS dependency or Node runtime client. Flags are fixed for the whole run, so a run never changes shape halfway through.

### 5.3 Executor (devplus-executor)

Job pods have no WorkOS credentials and must not get them.

- When the backend enqueues work (`deployment_queue`, `module_queue`, `health_queue`, `cost_sync_queue`, repo analysis, discovery), it stamps the executor-surface flags onto the queued document: `featureFlags: {"discovery-github-enrich": true, ...}`. This copies how `RepoAnalysisPolicy` freezes a policy onto each run (`devplus-shared/models/repo_analysis.go:208`).
- The dispatcher passes the field to the job pod (it already copies job fields into env/args). The job reads `job.FeatureFlags`, and if the field is absent it falls back to the existing env var. That keeps old queue items and a mixed-version rollout safe.
- `kill-deploy-apply`: the dispatcher re-checks it **at lease time** by reading the tenant snapshot from `tenant_settings.featureFlags` (Mongo, no WorkOS). An apply already queued is parked with a reason, not run. This is the one executor flag read at run time, because a kill switch that waits for the queue to drain is no kill switch.

### 5.4 MCP (devplus-mcp), CLI, SDK, TF provider

- **devplus-mcp** calls the backend with the user's token. At session start it reads `/auth/me` and hides tools whose `x-devplus-feature` is inactive from `tools/list` (`mcptools/server.go`). It does this per session, because a hosted MCP process serves many users. `mcp-access` inactive means the backend returns 404 for everything the MCP client forwards, so the hosted server answers with one clear error.
- **CLI** adds `devplus features` (table of slug, active, source), and maps `feature_unavailable` to "This feature isn't enabled for your workspace."
- **SDK / terraform-provider-devplus** pick up the typed error from the regenerated spec. A resource whose route is feature-gated reports the feature in its diagnostic.

### 5.5 Not affected

governance-service, oidc-service and opentofu-server get no flags. They are infrastructure behind the backend, and gating them remotely would break principle 3.

## 6. Timing and consistency

| Path | How long a flip takes |
|---|---|
| Backend route guard, schedulers | ≤ poll interval (30 s) |
| Frontend nav and pages | ≤ 5 min, or the next focus or reload |
| Agent run | next run (fixed for the whole run) |
| MCP tool list | next MCP session (backend 404s in the meantime) |
| Queued executor job | flags as of enqueue, except `kill-deploy-apply`, which is checked at lease |
| User-targeted flag, session caller | next token refresh (WorkOS claim semantics) |

## 7. Configuration

**Helm** (`devplus-platform/k8s/helm/devplus/values.yaml`, backend only):

```yaml
featureFlags:
  enabled: true            # false = the WorkOS layer is skipped, Fallback values apply
  pollSeconds: 30
  overrides: ""            # local/Tilt only; comma-separated slugs; rejected in prod
```

Backend env: `FEATURE_FLAGS_ENABLED`, `FEATURE_FLAGS_POLL_SECONDS`, `FEATURE_FLAGS_OVERRIDES`, plus the existing `WORKOS_API_KEY` and `WORKOS_ORG_ID`. Both of those are already in the backend pod, so no new secret or Terraform secret wiring is needed. Mirror the values in the `terraform-devplus-tenant/helm` chart copy, as the OTEL helper does.

**Terraform:** nothing required. An optional `terraform-devplus-tenant` variable `feature_flags = list(string)` could declare a tenant's entitlements next to its org, but see §8.1 for why a reconciler is preferred.

## 8. Operations

### 8.1 Targeting as code

The dashboard works for day-to-day flips. For entitlements that should be reviewed, keep `devplus-platform/feature-flags/targets.yaml`:

```yaml
ai-architect:
  dev:   { all: true }
  stage: { orgs: [app, test-stage] }
  prod:  { orgs: [] }
kill-repo-analysis:
  prod:  { orgs: [] }
```

A small reconciler (`devplus-platform/scripts/flags-sync`, Go, using `devplus-shared/featureflags/workos`) diffs that file against WorkOS: `GET /feature-flags`, then per slug `POST`/`DELETE …/targets/{org_id}` and `PUT …/enable|disable`. It resolves tenant names to org ids through `GET /organizations?…` (the org's `name` is the namespace, `terraform-devplus-tenant/main.tf:116`). It runs with `--plan` on PRs and applies on merge, using the per-env WorkOS key the deploy workflow already loads (`.github/workflows/_deploy-environment.yml:89`).

We don't use `restapi_object` for targets. The add-target call returns an empty body and has no per-target read, so the provider can't detect drift.

Emergency kills skip the file: flip in the dashboard, then follow up with a PR to the file (the reconciler's plan shows the drift until then).

### 8.2 CI checks

- devplus-shared: the registry has unique slugs that pass WorkOS's slug rule, and every `release` flag has a `RemoveBy`.
- devplus-platform (nightly and on PR): every registry slug exists in each WorkOS environment, and every WorkOS flag tagged `owner:*` exists in the registry. Orphans fail the job.
- Backend contract test (§4.4).

### 8.3 Testing

- `devplus-platform/tests/mocks/cmd/mock-workos/main.go`: add `GET /organizations/{id}/feature-flags` and `GET /user_management/users/{id}/feature-flags`, plus a test-only `PUT /_mock/feature-flags` to set them. Have the mock's `authenticate` put `feature_flags` into the tokens it mints.
- Regression tests: flag off → 404 `feature_unavailable`; flag on → normal; platform env off beats WorkOS on; WorkOS down with a stored snapshot → the snapshot is used; never answered → `Fallback`; a PAT caller gets user flags through the API path.
- E2E (Playwright): the nav item and page disappear when the mock flips a flag.
- `FlagService` unit tests: a fake clock, a fake transport, pagination, and backoff.

### 8.4 Lifecycle

1. Create the flag in the WorkOS dashboard (every environment), with tags `kind:` and `owner:`.
2. Register it in `devplus-shared/featureflags`, tag a shared release, and bump consumers.
3. Gate the code (`Feature:` on ops, `feature:` on nav, an agent or executor check).
4. Roll out: developer tenant → dev All → stage named orgs → stage All → prod named orgs → prod All.
5. Clean up (release flags): remove the gates, remove the registry entry, delete the flag in WorkOS. CI warns once `RemoveBy` has passed.

Update `devplus-docs/OPERATIONS.md` §kill switches to say: flag first (30 s, no deploy), env var second (deploy, survives a WorkOS outage).

## 9. Rollout plan

| Phase | Repos | Delivers |
|---|---|---|
| 0 | WorkOS dashboard, devplus-platform | Answer the open questions (§10); create the §3.3 flags |
| 1 | devplus-shared | `featureflags` registry, resolver, WorkOS REST client, JSON codegen; tagged release |
| 2 | devplus-backend | `FlagService` with snapshot, audit and OTEL; claims; `/auth/me` features; `/features`; `Op.Feature` with guard and contract test; scheduler checks; OpenAPI regen |
| 3 | devplus-platform | Helm values in both chart copies; mock-workos endpoints; regression and E2E tests |
| 4 | devplus-frontend | `useFeature`, nav and route gating, error mapping, admin panel; re-vendor the spec |
| 5 | devplus-agent-service, devplus-mcp | Agent subset and `kill-ai-writes`; per-session tool filtering |
| 6 | devplus-executor | Flags stamped on queue docs, job-side reads with env fallback, `kill-deploy-apply` at lease |
| 7 | devplus-platform, devplus-cli, devplus-sdk-go | Reconciler and CI, `devplus features`, OPERATIONS.md update |
| 8 (optional) | backend, governance-service | `context.features` in Cedar |

Phases 4-7 depend only on phase 2's API, so they can run in parallel. The existing env toggles stay in place throughout as the platform layer. None is removed until its flag has been in prod for one release.

## 10. Open questions

1. **WorkOS environments vs DevPlus environments.** Do dev, stage and prod use separate WorkOS environments (separate API keys and client ids), or one shared one? The MCP AuthKit issuer is shared today (`terraform-devplus-tenant/variables.tf:52`). Flag rules are per WorkOS environment. With one shared environment, "All in dev" isn't possible and everything is per-org targeting.
2. **The "enabled for org" list and `default_value`.** Check against the live API that a flag at rule All, or with `default_value: true`, appears in `GET /organizations/{id}/feature-flags`. §3.2's positive-only convention doesn't depend on the answer, but rule All does.
3. **Upgrading the Go SDK.** Do we schedule `workos-go` v4 → v10 (which brings `FeatureFlags()` natively), or keep the thin REST client indefinitely?
4. **Per-project rollout.** Do we ever need it? If so, the options are a Node sidecar running the runtime client with custom `project` targets, or a project-level layer in `tenant_settings`. Leaving it out keeps the Go backend as the single evaluator.

## Appendix: today's toggles, by location

- Helm: `values.yaml` lines 64, 101, 132, 180-182, 213-214, 230, 236, 314, 329, 480, 503; `PLAN_GATE_FAIL_MODE` hardcoded at `templates/executor-job-dispatcher.yaml:111`.
- Backend: `services/schedule_policy.go:67-96`, `cost_service.go:53-64`, `health_schedule.go:45`, `preview_service.go:44`, `repo_analysis_service.go:137`; scheduler start at `internal/app/app.go:620-668`; tenant settings routes `internal/server/routes.go:620-690`.
- Executor: `services/repo_analysis_job.go:138`, `provider_tags.go:111,123`, `job_dispatcher.go:162`, `health_job_mongo.go:272`, `discovery_job.go:773`, `plan_gate_client.go:51`.
- Agent service: `api/models/settings.py:32,47`; graph `domain/graph/graph.py:191-201`.
- Governance: `internal/app/app.go:48-50`, `services/governance_service.go:62-68`.
- MCP: `internal/service/server.go:49`.
- Terraform: `terraform-devplus-tenant/variables.tf:44,188,194,251,270,283`; `terraform-helm-devplus-services/variables.tf:244,295,308`.

## Sources

- https://workos.com/docs/feature-flags
- https://workos.com/docs/feature-flags/access-token-claim.md
- https://workos.com/docs/feature-flags/node-runtime-client
- https://workos.com/docs/feature-flags/custom-targets.md
- https://workos.com/docs/reference/feature-flags/targeting/list-for-organization.md
- https://workos.com/docs/reference/feature-flags
