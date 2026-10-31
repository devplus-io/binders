---
id: 98c92623-b961-4171-a05f-cea267a42bce
binder: binder-mvvle32a-1
---
State Versioning, Diffs, and Rollback

Reference specification for **V0.18** (inspired by Terrakube's state history and visual diff capabilities).

---

## 1. Problem & Motivation

Today, `devplus-opentofu-server` (`services/state_service.go` → `mongoStateStore.UpsertSealed`) stores a single document per `(project, name)` in `terraform_states`, overwriting the `sealed` state blob in place on every `POST /state/{project}/{name}`.

- **No historical audit or recovery**: If an apply fails partway through, a module upgrade mutates state unexpectedly, or a brownfield adoption/refinement writes an undesired state, the prior working state is overwritten with no way to inspect what changed or roll back.
- **Unchecked unlocked writes**: `HandleStateUpdate` in `devplus-opentofu-server/handlers/state_handler.go` only verifies the lock when `?ID=<lockID>` is passed in the query string; if `?ID=` is omitted while a lock is held, the write is not rejected.

---

## 2. Architecture & Security Invariants

1. **Envelope encryption at rest**: Every historical state snapshot in `terraform_state_versions` is sealed with `devplus-shared/crypto` (`EnvelopeSealer`, `scope = tenant DB name`), identical to `terraform_states`.
2. **Zero plaintext secrets outside state**: State diffs are computed server-side in `devplus-backend` and scrubbed through `ScrubCredentialValues`, `IsCredentialName`, and OpenTofu's `sensitive_attributes` paths before returning to any caller (SPA, CLI, SDK, MCP, or AI agent). Raw state JSON is never exposed on public API routes.
3. **Project boundary enforcement**: All backend routes under `/projects/{projectId}/deployments/{id}/state-*` are wrapped in `RequireProjectResource` (`Deps.ProjectScope.Deployment`), returning `404` when the deployment belongs to another project.
4. **Person-in-the-loop + step-up auth for rollback**: State rollback requires a browser session (`context.authMethod == "session"`, enforced by a seeded Cedar forbid matching `ApproveProjectDeployment` and `RevealDeploymentOutput`) with an active sign-in within the last 15 minutes (`SessionAuthAt`, returning `403 reauth_required` otherwise).
5. **Fail-closed auditing**: Every state rollback claims an `audit_events` row (`RollbackDeploymentState`) before mutating state; if the audit write fails, the rollback is refused (`503`).

---

## 3. Changes by Repository

### 3.1 `devplus-opentofu-server`

- **State Header Extraction on Save (`services/state_service.go`)**:
  - In `SaveState`, parse the lightweight OpenTofu v4 state header (`version`, `serial`, `lineage`, `terraform_version`, and managed resource instance count) before sealing `data`.
- **Tighten Lock Verification (`handlers/state_handler.go` & `services/state_service.go`)**:
  - If the current `stateDocument` has a non-nil `Lock`, reject `SaveState` with `423 Locked` whenever `lockID == ""` or `lockID != doc.Lock.ID`.
- **New `terraform_state_versions` Collection (`services/state_store.go`)**:
  - Document shape (`stateVersionDocument`):
    - `project` (`string`), `name` (`string`)
    - `serial` (`int64`, from tfstate or monotonic fallback)
    - `lineage` (`string`)
    - `tofu_version` (`string`, from `terraform_version`)
    - `managed_resource_count` (`int`)
    - `lock_id` / `operation` / `who` (captured from the active `LockInfo` at write time)
    - `restored_from_serial` (`*int64`, set when created by a rollback)
    - `sealed` (`models.SealedSecret`)
    - `created_at` (`time.Time`)
  - Indexes: unique compound index on `(project, name, serial)` (`{project: 1, name: 1, serial: -1}`).
  - **Retention & Cleanup**: Retain the newest `MAX_STATE_VERSIONS` (default `20`) per `(project, name)`, pruning older serials after insert. When `DeleteState` (`DELETE /state/{project}/{name}`) is called after a successful destroy, delete all versions for `(project, name)`.
- **New Internal Routes (`internal/server/routes.go`, guarded by `requireInternalToken`)**:
  - `GET /state/{project}/{name}/versions` — Returns metadata list (`serial`, `lineage`, `tofuVersion`, `managedResourceCount`, `operation`, `restoredFromSerial`, `createdAt`) ordered by `serial DESC` without decrypting state blobs.
  - `GET /state/{project}/{name}/versions/{serial}` — Unseals and returns the raw state JSON for a specific `serial` (`404` if not found).
  - `POST /state/{project}/{name}/rollback` (body `{"serial": <targetSerial>}`) — Verifies no lock is held (`423 Locked` if locked), unseals `targetSerial`, rewrites the JSON `serial` field to `currentMaxSerial + 1` (so OpenTofu's HTTP backend accepts subsequent writes without serial-regression errors), seals and upserts it as the head in `terraform_states`, and inserts the new revision (with `restored_from_serial = targetSerial`) into `terraform_state_versions`.

---

### 3.2 `devplus-shared`

- **Models (`models/state_version.go`)**:
  - `DeploymentStateVersion`: `{serial, lineage, tofuVersion, managedResourceCount, operation, restoredFromSerial?, isCurrent, createdAt}`
  - `DeploymentStateVersionList`: `{deploymentId, currentSerial?, versions []DeploymentStateVersion}`
  - `StateResourceChange`: `{address, type, name, provider, action (added|changed|removed), attributeDiffs []AttributeDiff}`
  - `StateOutputChange`: `{name, action (added|changed|removed), before, after, sensitive}`
  - `DeploymentStateDiff`: `{deploymentId, fromSerial, toSerial, summary{added, changed, removed}, resources []StateResourceChange, outputs []StateOutputChange}`
  - `RollbackDeploymentStateRequest`: `{serial int64, comment string}`
  - `RollbackDeploymentStateResponse`: `{deploymentId, restoredFromSerial, newSerial, managedResourceCount, restoredAt}`
- **Cedar Actions (`models/actions.go`)**:
  - `ListProjectDeploymentStateVersions` (Read tier)
  - `GetProjectDeploymentStateDiff` (Read tier)
  - `RollbackProjectDeploymentState` (Project Admin tier; browser-only session forbid in `internal/seed/`)
- **Audit Constants (`models/audit_event.go`)**:
  - `AuditActionRollbackDeploymentState = "RollbackDeploymentState"`, `AuditTargetDeploymentState = "deployment_state"`

---

### 3.3 `devplus-backend`

- **Opdef Routes (`internal/server/routes.go`)**:
  - `GET /projects/{projectId}/deployments/{id}/state-versions` (`ListProjectDeploymentStateVersions`, wrapped in `scope.Deployment`)
  - `GET /projects/{projectId}/deployments/{id}/state-diff?from=<serial>&to=<serial>` (`GetProjectDeploymentStateDiff`, wrapped in `scope.Deployment`)
  - `POST /projects/{projectId}/deployments/{id}/state-rollback` (`RollbackProjectDeploymentState`, wrapped in `scope.Deployment`)
- **Scrubbed State Diff Service (`services/deployment_state_version_service.go`)**:
  - Fetches `fromSerial` and `toSerial` raw states from `opentofu-server`.
  - Matches resource instances by canonical address (`[module.]type.name[index]`) and diffs attributes.
  - Redacts any attribute covered by OpenTofu `sensitive_attributes`, `sensitiveOutputs`, `IsCredentialName`, or `ScrubCredentialValues` to `"(sensitive)"`.
- **Rollback Execution & Post-Rollback Sync**:
  - Checks `SessionAuthAt` (15-minute step-up window; returns `403 reauth_required` if stale).
  - Checks `DeploymentQueueService.ActiveActions(ctx, deploymentID)` — returns `409 Conflict` if any deploy, delete, drift check, adoption, or refinement is queued or running.
  - Claims an `audit_events` row (`pending` → `done`/`failed`) before calling `opentofu-server`.
  - Calls `POST {opentofu-server}/state/deployments/{id}/rollback`.
  - Re-extracts `Deployment.resources` and masked `Deployment.outputs` (`outputs_version: 2`) from the restored state so the MongoDB `Deployment` document is immediately consistent with the rolled-back state.

---

### 3.4 `devplus-frontend`

- **Deployment Detail Page (`/projects/:id/deployments/:id/logs`)**:
  - Add a **State History** card/tab showing the timeline of state serials (`Serial #N`, timestamp, OpenTofu version, managed resource count, and a `Restored from #M` badge when applicable).
  - **Compare Versions Drawer**: Select any two serials (defaults to comparing a selected serial against its predecessor) to inspect `+ added`, `~ changed`, and `- removed` resources and attribute diffs with `(sensitive)` masking.
  - **Restore State Version Modal (Project/Tenant Admins)**: Confirmation modal with optional comment, handling the `403 reauth_required` step-up redirect flow and offering a one-click **Check drift now** after restore.

---

## 4. Acceptance / Done Criteria

- [ ] Saving state in `devplus-opentofu-server` records a sealed version in `terraform_state_versions` and prunes versions beyond the retention cap (`20`).
- [ ] `POST /state/{project}/{name}` while a lock is held without a matching `?ID=<lockID>` returns `423 Locked`.
- [ ] `GET .../state-versions` and `GET .../state-diff` return version metadata and scrubbed resource/output diffs with zero plaintext secrets exposed.
- [ ] `POST .../state-rollback` enforces project scope (`404`), browser session + 15m step-up (`403`), no active queue item or state lock (`409`), and fail-closed `audit_events` logging (`503`), increments `serial` to `max + 1`, and syncs `Deployment.resources` and `Deployment.outputs`.
- [ ] Whole-repo test coverage remains $\ge 80\%$ across `devplus-opentofu-server`, `devplus-backend`, and `devplus-frontend`.
