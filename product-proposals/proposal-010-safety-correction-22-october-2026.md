---
id: 4e0b2120-4909-48d0-b2b3-7b70a4cb4742
binder: binder-mvvle32a-1
folder: folder-mvw0u8o0
---
Proposal 010 safety correction, 22 October 2026

Source: ~/Documents/DevPlus Architecture/proposals/010-drift-remediation.md; read-only mirror backend 6b31679, executor 74743d2, shared fdf993d, frontend 7e66098

Proposal 010 remains an unsubmitted design, not an owner decision or filing approval. No founder greenlight/customer demand was verified. No code or GitHub issue was created.

Direct source findings: drift parser services/drift_diff.go:90-101 creates display resource actions without digests; the deploy parser services/plan_changes.go:78-108 computes raw-change digests. services/executor_service.go:2050-2059 can return drifted with a nil summary after JSON read failure. Therefore the stored drift summary is not a plan/approval artifact. Executor deploy_approval.go:62-153 compares the current configuration fingerprint and checks every new change against the approved digest set, permitting a subset rather than exact equality; DeployWithoutApproval can bypass requesting a human. Backend queue Enqueue unconditionally upserts status/lease, so live work must not be protected by a read-then-enqueue guard alone. Shared environments.go:278-350 distinguishes last applied configuration from a pending/rejected change.

Corrected spec: completed non-preview non-adopted deployments only; browser Review repair plan; new project-scoped prepare route authorized as a write, not as drift read; mandatory fresh-plan approval through existing browser session and second-person rules; operation-aware concurrency reservation, idempotency and durable delivery; exact repair-specific plan/config/source binding; fail-closed governance; correlated post-apply verification, with successful apply separate from verified convergence. Agent tool deferred. Estimate 8-12 engineering days, provisional pending Mongo transaction and immutable-source validation. Chart defaults do not establish tenant usage. No running tenant release was verified. Parked advisor and issue #71 decisions remain untouched.
