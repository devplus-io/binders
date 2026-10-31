---
id: fddc87db-d023-46ca-b06e-dc21227194ad
binder: binder-mvvle32a-1
folder: folder-mvw0u7ti
---
Architecture review 2026-10-26

The weekly review refreshed all 32 active DevPlusInc repository mirrors, regenerated the map and ran the health scan. Compared with the 19 October raw snapshot, open Dependabot alerts rose from 21 to 30: 7 critical, 11 high, 11 medium and 1 low. The 20 October raw snapshot has 22 alerts; older memory saying 21 for that date was incorrect.

Priority improvements already have open issues, so no new duplicate issues were filed:
1. Project writer-to-admin escalation: https://github.com/DevPlusInc/devplus-backend/issues/94. Main handlers/project_group_handler.go:59-105 passes requested permissions to the service, and internal/seed/seed.go:397 grants the group mutation actions to project writers.
2. Service-token identity claims: https://github.com/DevPlusInc/devplus-oidc-service/issues/42. Main internal/token/service.go:174-201 merges caller claims after reviewed identity. Important fix detail: applyAdditionalClaims at :99-112 permits non-system sub replacement for project/deployment federation, so it is insufficient unchanged for service tokens.
3. Provider dependency refresh: https://github.com/DevPlusInc/terraform-provider-devplus/issues/3. All 7 critical alerts are here, plus 5 high and 5 medium. go.mod and tracked vendor/modules.txt still pin x/crypto0.51.0 and grpc1.79.3. CI and releases build with -mod=vendor; latest actual provider CI is from July, despite recent green CodeQL runs.

Three comments were posted and independently read back, with exact body and penant-agent[bot] authorship verified:
https://github.com/DevPlusInc/devplus-oidc-service/issues/42#issuecomment-6417773917
https://github.com/DevPlusInc/terraform-provider-devplus/issues/3#issuecomment-6417775159
https://github.com/DevPlusInc/devplus-agent-service/issues/59#issuecomment-6417776639

The agent issue follow-up corrects the old blocker: main now locks pymongo4.18.2 and langgraph-sdk0.4.5, the main build succeeded, PR58 closed without merge, and neither old advisory appears among open agent-service alerts. The remaining agent alert concerns Black tooling. Two other high-alert pins disagree with main: governance compress is1.19.2 and core callback grpc is1.83.2. These are raw alert metadata, not proof of production exposure.

Latest main repository-level sampled runs are green for19 repos;13 have no run in that sample. All8 current service build workflows found in the detailed sample are green. Platform test matrix is green. File counts:1317 non-test source files and847 test files, up330 and319 respectively; this is not line/branch coverage. Dependency lag sample:50 entries, up5. Markers:8, unchanged. Generated frontend API declarations and placeholder strings explain some large-file/marker signals; no issue was filed just for size.

Chart0.1.17 was published25 October and release source is merged. CHANGELOG still puts its notes under Unreleased. Current tenant versions could not be verified because dev, stage and prod Kubernetes APIs timed out; no production or network changes were made. Source-backed docs must not imply verified runtime state.

Updated the Binders product brief and created the curated repository map. Updated the local detailed brief and diagram captions. Rendered and visually checked hub-staff-network PNG/SVG with the Binders private endpoint and Central US PostgreSQL server; Binders is separate from DevPlus tenant MongoDB. Member/RLS and Covee software policy were not checked.

Evidence files: ~/Documents/DevPlus Architecture/health/2026-10-26.json and health/review-evidence/2026-10-26-*. Raw source baseline19 October; GitHub metadata checked26 October. Helper result waits timed out; their conclusions were not used. The reported code evidence was independently read from the mirror.
