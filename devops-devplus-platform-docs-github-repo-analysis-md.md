---
id: 63b2ebd4-0fbf-44c2-8eb2-b78adb5795db
binder: binder-mvvle32a-1
---
DevOps | devplus-platform/docs/GITHUB_REPO_ANALYSIS.md

Source: DevPlusInc/devplus-platform/docs/GITHUB_REPO_ANALYSIS.md, branch docs/devops-refresh-2026-10, refreshed 2026-10-04

Refreshed 2026-10-04 against origin/main code (DevPlusInc/devplus-platform, docs/GITHUB_REPO_ANALYSIS.md).

# GitHub Repository Analysis: Operations Runbook

Design: `devplus-workspace/outputs/github-repo-analysis-plan.md`. Everything
below describes what the code does, verified by running it against a real
GitHub organization (inventory, a single-repository analysis and a full
32-repository run) on an AKS tenant.

## What it does

Two passes over a connected GitHub App installation.

**Pass A, inventory.** One job pod per connector on the existing discovery queue
(`JOB_KIND=discovery`, `CONNECTOR_TYPE=github`). No clone. Three tiers:

1. *List*: `/installation/repositories`, paged. Identity for every repository.
2. *Bulk*: one GraphQL query per 25 repositories returns the default branch's
   head commit, `isEmpty`, `isTemplate`, the real last-commit date, exact
   branch / tag / release / commit totals and the 50 most recent releases, for
   **every** repository, at about one rate-limit point per batch.
3. *Enrich*: per-repository REST lists (branches with their protection flag,
   contributors), following `Link` pagination, for the most recently pushed
   repositories while the budget lasts (default 200 repositories, 3000 calls).

It writes `github:*` inventory rows, the durable `github_repo_ledger` (one row
per repository, including the API half of its list digest) and one
`repo_analyses` document per repository with `source:"api"` plus its `git` and
`authors` facet chunks. A repository the installation no longer covers is
retired from the ledger, with its analyses. A rate limit or a rejected
credential **stops** enrichment; the repositories not reached stay at the
`cheap` tier and say so.

**Pass B, content.** One job pod per repository (`JOB_KIND=repo_analysis`, job
name `repoanalysis-<ledgerId>`). It asks the backend for a read-only token
scoped to that one repository, shallow-clones into a quota'd emptyDir and runs
deterministic filesystem analyzers (languages, dependencies with a CycloneDX
SBOM, vulnerabilities via OSV, licenses, symbols, imports, API surface,
architecture and classification, IaC, CI, tests, secrets, risk).

Those analyzers parse the real files: manifests and lock files per ecosystem
(so a component tree is the resolved one, with direct and transitive counted
separately), Go syntax trees for routes and symbols, and per-framework readers
for Express, Python, OpenAPI, Terraform, Helm, Kubernetes, compose and workflow
files. Two consequences worth knowing: a repository's component count is its
full resolved tree, not its direct dependencies; and a route found in a SCHEMA
document (a vendored `openapi.json`) does not make the repository a service,
because a client vendors its server's spec. It writes a
`source:"fs"` `repo_analyses` document, chunked `repo_analysis_facets`, and
GridFS artifacts in the `repo_analysis` bucket. No model is called anywhere.

A repository's profile is the latest completed document from each source. They
are separate documents with separate lifecycles on purpose.

## Triggers and the run state machine

| Call | What it does |
|---|---|
| `POST /api/v1/github/connectors/{id}/analysis` | Installation-wide run under a selection policy. Forks, archived and empty repositories are left out unless the policy opts them in. `409` while repositories from an earlier run are still queued or analyzing. |
| `POST /api/v1/github/connectors/{id}/repositories/{repoId}/analysis` | Exactly one repository. Runs for forks and archived repositories too, forces a re-analysis by default (`{"force": false}` for only-if-changed), touches no other repository. `400` for an empty or oversized repository, `409` if that repository is already in flight. |
| `POST .../analysis/cancel` | Cancels every active run of the connector and drops queued items. Pods already running finish; their terminal writes are guarded. |
| `GET .../analysis` | The connector's latest run and its counters (the read the Repositories tab polls). Closes orphaned runs first. |

Ledger state per repository: `never -> queued` (backend, at enqueue) `-> running`
(dispatcher, when the Job is created) `-> completed | failed | skipped`. A pod
that finds the repository already fresh, or its run cancelled, returns the row
to what its stored result implies (`completed`, or `never`). Only a durable
property of the repository (empty, too large) leaves it `skipped`.

Run counters: `reposQueued` is what was dispatched; `reposCompleted`,
`reposPartial`, `reposFailed` and `reposSkipped` are pod outcomes and are the
only terms that close a run; `reposExcluded` / `excludedByReason` are what the
policy left out at trigger time. The dispatcher moves the run to `running` on
the first Job; the last finishing pod (or the dispatcher, when the last
repository fails out of its retry budget) closes it. A run with nothing left in
the queue is closed the next time anyone reads the latest run.

Freshness: an unchanged head commit, the same engine version and the same
content fingerprint (depth, facets, per-repository file ceiling) means the pod
skips before cloning. Selection terms are not part of that fingerprint, so
analyzing one repository alone and then the whole installation does not
re-clone it.

The engine version is `models.RepoAnalysisEngineVersion`, currently
**`repo-analysis-v2`**. Bump it whenever an analyzer's output contract changes;
the next run then re-analyzes every repository, because a stored result from a
weaker analyzer is not the same fact. It lives in `devplus-shared` so the
backend (which decides freshness) and the executor (which stamps results)
cannot disagree.

## What is analyzed: the default branch

Pass B clones the repository's **default branch** at its head commit. A
repository whose work lands on another branch is therefore described by
whatever its default branch holds. Two of this organization's own repositories
were exactly that case when this was verified: `devplus-hermes-agent` and
`devplus-hermes-state` (both retired on 2026-09-23) worked on `spark` and left
`main` deliberately stale, so their analysis reported "0 components, no CI, no
tests" and a high risk score for a `main` that genuinely contained none of it.
The facts are right about the branch that was read.

Analyzing a chosen branch is not built. Until it is, read a repository's
`defaultBranch` before drawing conclusions from its digest.

## Reading results

| Surface | Routes | Who |
|---|---|---|
| Connector (admin) | `/api/v1/github/connectors/{id}/repositories[...]`, `/analyses/{id}[/facets/{facet}\|/artifacts/{facet}]` | Tenant admins. The Discovery pages use these: GitHub connectors are created tenant-wide by the App install flow and have **no owning project**. |
| Project | `/api/v1/projects/{pid}/github/...` | Project members and agents (agent-service's read-only `repo_profile` and `repo_facts` tools), constrained to connectors granted to the project. Secret findings are their own route and action (`.../analyses/{id}/secrets`, write tier); the generic facets route refuses `facet=secrets`. Artifact download is write tier too. |

Lists use an opaque keyset cursor that is only valid under the `sortBy` it was
issued for.

## The ecosystem graph (Insights)

`GET /api/v1/github/connectors/{id}/ecosystem-graph` joins every repository's
recorded facts into one graph over the installation, and the **Insights** tab
draws it. Node kinds: `repository`, `contributor`, `library`, `module`,
`external` (a module source outside the installation) and `language`. Edge
kinds: `contributes` (person to repository, with commits), `depends-on`
(repository to library, with the pinned version), `calls` (repository or
module to module, with how the call is pinned) and `written-in` (repository to
language, with the share of code).

Where each kind comes from, and the rules that keep the picture readable:

- **Committers** come from the inventory pass (the `authors` facet of the
  api-source analysis), so they exist for repositories nobody has analyzed
  yet. One GitHub login is one node however it was capitalized; an author the
  inventory could only see as an email hash is counted in `unresolvedAuthors`
  and not drawn. Bots are flagged and hidden by default in the tab.
- **Libraries** are direct dependencies only (a transitive component is a fact
  about a lockfile, not about what the code imports), and only those at least
  two repositories share; a library one repository uses is on that
  repository's own page and counted in `librariesSingleRepo`. The node carries
  every distinct version in use, which is the "who is on which version"
  question, capped at 500 libraries by reach.
- **Languages** link to a repository when they are its primary language or at
  least 1% of its code, and only programming and markup languages count: a
  0.2% Shell footprint is a Makefile, and Markdown is prose.
- **Modules** are the Terraform module graph folded in. Sources resolve the
  way Terraform reads them: a local source stays inside its repository, a git
  source whose path names a repository in this installation resolves to it
  (and to a subdirectory module when the source has a `//` part), and anything
  else becomes one external node per module, so two callers on different
  versions share a node and the versions read as two edges. A repository
  whose root is the module is one node carrying both identities. Each call is
  classified by how firmly it is pinned: `commit`, `tag`, `branch`,
  `constraint` (a registry version range), `none` or `local`. A `branch` or
  `none` is the finding worth acting on, because what the next apply fetches
  can change with no commit in the calling repository.

Every node carries its `degree` and, for anything but a repository, the number
of distinct `repositories` it reaches, so a client can rank and size before it
lays anything out. The response is capped at 4,000 nodes; repositories are
never dropped, the least connected of the other kinds are, and `truncated`
says so.

The graph names what it could not see. A repository with no content analysis
is still a node, because it can be the target of a module call and dropping it
would silently drop that edge, but it is listed in `notAnalyzed`;
`reposWithAuthors` says how many repositories the inventory listed committers
for. The tab lays the graph out client side (a deterministic force layout, so
the same installation always draws the same way) and lets the reader pan,
zoom, drag nodes and search.

## A repository's architecture

The Architecture section of a repository's page reads the `arch` and
`imports` facets in four views. **Map** draws the deployable units, the code
layers as bands proportional to their lines, and the external systems the
code reaches, with one line per system styled by how the engine knows it (an
import or manifest is solid, an environment variable name dashed, a literal
host dotted). **Dependencies** draws the repository's own import graph: Go
packages under the module path, or directories for file-level languages, at
a depth the reader picks, with cycles in red. **Components** and
**Classification** list the units and the verdict with its evidence.

Color on all of these surfaces follows one rule, defined in the frontend's
`lib/subjects.ts`: one hue per kind of thing, the same wherever it appears. A
repository is the brand blue, a person magenta, a library orange, a Terraform
module teal, a language violet, a deployable unit cyan, an API amber, tests
olive and a CI pipeline blue, on stat cards, section marks, chart bars, graph
nodes and graph edges alike; a language keeps a hue close to its own
community's (Go cyan, HCL purple) on every chart and table. Status colors
(success, warning, danger) keep their meaning and are never borrowed for a
kind, so a red ring or a warning badge is always a finding. People are shown
with their GitHub avatar, fetched from github.com by login.

## Helm

`repoAnalysis.enabled` (default **true**) is rendered as
`REPO_ANALYSIS_ENABLED` on both the dispatcher and the backend. When false the
dispatcher does not lease analysis work and the backend answers both triggers
with `503`; stored results stay readable. Nothing runs until an admin presses
Analyze either way.

| Value | Env | Default | Meaning |
|---|---|---|---|
| `serviceAccountName` | `REPO_ANALYSIS_SERVICE_ACCOUNT` | `""` (= `<ns>-repo-analyzer`) | the bare ServiceAccount the chart creates for the static-credentials case; unused under MONGODB-OIDC (below) |
| `jobMemoryLimit` / `jobMemoryLimitFull` | `REPO_ANALYSIS_JOB_MEMORY_LIMIT[_FULL]` | `1Gi` / `2Gi` | pod limit, shallow / full clone |
| `jobMemoryRequest` | `REPO_ANALYSIS_JOB_MEMORY_REQUEST` | `512Mi` | pod request |
| `jobCpuLimit` | `REPO_ANALYSIS_JOB_CPU_LIMIT` | `1` | |
| `scratchLimit` | `REPO_ANALYSIS_SCRATCH_LIMIT` | `2Gi` | emptyDir `sizeLimit` and ephemeral-storage request. The pod budgets itself to 80% of it, so an oversized repository is skipped as `too-large` instead of being evicted and retried. |
| `maxParallelPerConnector` | `REPO_ANALYSIS_MAX_PARALLEL_PER_CONNECTOR` | `3` | unfinished Jobs per connector |
| `jobLeaseMinutes` | `REPO_ANALYSIS_JOB_LEASE_MINUTES` | `8` | the pod heartbeats at lease/3, so this is how long a repository stays `running` after its pod dies, not a run-time limit |
| `jobMaxAttempts` | `REPO_ANALYSIS_JOB_MAX_ATTEMPTS` | `3` | a concurrency deferral does not spend an attempt |
| `activeDeadlineSeconds` / `...Full` | `REPO_ANALYSIS_ACTIVE_DEADLINE[_FULL]_SECONDS` | `300` / `900` | pod wall clock |
| `keepRuns`, `retentionDays` | `REPO_ANALYSIS_KEEP_RUNS`, `REPO_ANALYSIS_RETENTION_DAYS` | `3`, `90` | both reach the pod that enforces them |
| `osvApiBase` | `OSV_API_BASE` | `https://api.osv.dev` | empty turns the vulnerabilities facet **off**: reported `not-configured`, never clean. Job pods need egress to this host. |
| `inventoryMaxApiCalls`, `inventoryMaxEnrichRepos` | `GITHUB_INVENTORY_MAX_API_CALLS`, `GITHUB_INVENTORY_MAX_ENRICH_REPOS` | code: 3000, 200 | read by the **discovery** pod |

The dispatcher Role has `delete` on `batch/jobs` (stale finished Jobs are
removed before a repository is re-analyzed; Job names are keyed by ledger row).

## The pod, and what it can reach

Hardened container: uid 65532, read-only root filesystem, no capabilities,
seccomp `RuntimeDefault`, no Kubernetes API token, a quota'd scratch volume
(`/work`, which is also `HOME`; `TMPDIR=/work/.tmp` because ctags spools to a
temp file), and a wall-clock deadline. The executor binary lives in
`/usr/local/bin` because `/root` is not readable by an unprivileged user.

Identity follows database auth:

- **Static Mongo credentials** (local k3d, atlas mode): the dedicated,
  binding-less `<ns>-repo-analyzer` ServiceAccount.
- **MONGODB-OIDC** (AKS): `<ns>-app` with the workload-identity label, because
  Atlas has a database user only for that account. The signal is the
  connection string's mechanism, not `EXECUTOR_JOB_WORKLOAD_IDENTITY` (which is
  true in every Tilt mode and on every Atlas tenant).

Stated plainly, a compromised analysis pod can reach the tenant's MongoDB and
the backend's internal API through `INTERNAL_API_TOKEN`, the same shared token
every other job pod holds. It never holds the GitHub App private key, and the
backend mints GitHub credentials only for work the dispatcher has leased (an
in-progress analysis item for that repository; an in-progress discovery item
for the inventory token).

Secret findings never carry the matched value: a finding is a path, a line, a
rule, a preview of at most four characters, an entropy figure and a
fingerprint. The fingerprint is salted with the tenant's database name, which
is not itself a secret, so it prevents cross-tenant correlation but does not
make a LOW-entropy match unguessable to somebody holding a dump of the tenant's
facets. Do not describe it to a user as unrecoverable.

Open hardening, in order of value: a per-job ticket instead of the shared
internal token; an egress NetworkPolicy for `kind=repo_analysis` pods (backend,
DNS, Mongo, `github.com`, `api.osv.dev`); a key the database does not contain
for the secret fingerprint (above); a dedicated Atlas user for the
analyzer account so AKS can use the dedicated ServiceAccount too (a projected
token with audience `api://AzureADTokenExchange` is enough; Atlas trusts the
cluster issuer directly, no Azure identity needed). That last one lives in
`terraform-devplus-tenant` and the Tiltfile.

## GitHub App permissions

No new App permissions. The App already carries `contents:write`; the analysis
token is requested at `contents:read`, and the grant GitHub echoes back is
verified before the token leaves the backend: no unrequested permission, no
higher level, exactly the requested repository. `security_events`,
`pull_requests` and `members` are requested at read only when the installation
holds them.

Whether the installation covers all repositories or a selected subset is read
from the installation itself and stamped on the run. (A repository-scoped token
always says "selected", which is not the same fact.)

## Quotas

About 5,000 REST requests per hour per installation, shared with module scan,
catalog import and the Architect's repository writes. Pass A's bulk tier costs
points, not requests. A 32-repository organization inventories in about 70
requests. Pass B spends one token mint per repository and no GitHub API calls.

## Troubleshooting

- **A repository stays `queued`.** Three repositories per connector run at a
  time; the rest wait. `kubectl -n <tenant> get jobs -l kind=repo_analysis`.
- **A re-analysis sits at `0/N` for about a minute before the first pod.**
  Expected, once: Job names are keyed by ledger row, so the previous run's
  finished Job still holds the name until its TTL. The dispatcher deletes each
  one and puts the item back with a short delay, which costs one cycle per
  repository before any new pod starts. The log says
  `deleting stale finished repo-analysis job …` for each.
- **A repository stays `running`.** Its pod died. The lease expires
  (`jobLeaseMinutes`), the dispatcher reclaims it, and past `jobMaxAttempts`
  marks it `failed` and closes the run if it was the last one.
- **`Analysis already in progress` (409).** Repositories from an earlier run
  are still queued or analyzing. Wait, or cancel that run.
- **`No job in progress for this credential` (409) in a pod log.** The pod
  asked for a GitHub credential with no leased queue item behind it. Expected
  only if the item was cancelled between dispatch and the mint.
- **Vulnerabilities say `not-configured`.** `osvApiBase` is empty. `failed` or
  `partial` means the pod could not reach OSV: check egress.
- **Vulnerabilities say `skipped: no-resolved-components`.** No lockfile was
  parsed, so there are no concrete versions to look up. Check the sca facet's
  manifest rows: each one carries a `parseQuality`, which names the lock file
  and whether it was read.
- **Inventory reports scan errors mentioning a rate limit.** Enrichment stopped
  on purpose. Every repository still has its head commit and counts; run
  discovery again after the reset for the branch and contributor lists.
- **Artifacts 404.** Only references recorded on the analysis document are
  served. `503` means the GridFS bucket is not wired in that deployment.

## Retention and deletion

Each repository keeps its newest `keepRuns` content analyses, none older than
`retentionDays`; facet chunks and GridFS files go with their analysis document.
There is one API profile document per repository, overwritten by each inventory.

Deleting a GitHub connector purges everything derived from it, in this order:
active runs are cancelled, the queue is emptied, GridFS artifacts are deleted
(the analysis documents are their only index, so they go first), then facets,
analyses, runs and the ledger, then the discovery queue item, inventory rows and
result document. If the purge fails the connector is not deleted; deleting it
again retries.

## Release ordering

`devplus-shared` is released by git tag, and backend and executor `require` it.
Cut the shared tag first, bump both `require` lines together (verify with
`GOWORK=off`, which is what CI runs), then backend and executor, then frontend
(its CI diffs the generated `schema.d.ts` against the vendored
`openapi/openapi.json`, already updated on the branch), then agent-service and
this repo. The executor image change (binary in `/usr/local/bin`) and the chart
change ship together in the same release. The feature first shipped this way in
platform release 0.1.15 (devplus-shared v1.15.0; see `CHANGELOG.md`).
