---
id: a19962a9-0038-4a5b-beda-801f45897a8c
binder: binder-mvvle32a-1
---
SRE report · 2026-09-24-ops-summary

# DevPlus Ops — recent work, 23–24 September 2026

**Interview summary. Evidence checked through 24 Sep 08:54 UTC. No changes made for this interview.**

## At a glance

- **Infrastructure:** hourly monitoring and incident investigations across five AKS clusters.
- **Incident notes:** nine incident files are present in `reports/`.
- **Production changes:** none during the documented checks; no release, tenant deployment, rollback or infrastructure mutation was performed.
- **Support:** support log records four tickets handled on 23 Sep; the queue showed zero open when queried during this interview. The query's result was not written to the support log.

## Completed checks and actions

| Date | Work | Evidence / result |
|---|---|---|
| 23 Sep | Full read-only infrastructure check | All five AKS clusters reachable; 26/26 nodes Ready, no unhealthy or pending pods. Report: `2026-09-23.md`. Cloudflare and MongoDB Atlas were not checked because credentials were unavailable. |
| 23 Sep | Hourly monitoring and incident investigations | Reviewed transient stage rollout/CNI warnings and node churn in prod-eus2, prod-cus and stage-cus. Workloads recovered; no customer-workload impact observed. See dated entries in `log.md` and the incident notes in this folder. |
| 23 Sep | Provisioning and change-path investigation | Traced `NoCompatibleInstanceTypes` to general NodePool constraints and found the shared Helm chart default `consolidateAfter: 1m`. Found prod/stage Terraform has `deploy_aks=false`; turning it on risks a plan to create clusters. No apply or patch was run. |
| 24 Sep, 00:15–03:15 UTC | Monitoring-agent OOM and node churn investigation | Confirmed `ama-logs-rs` OOMs self-recovered and were AKS-addon-managed, not tenant workload failures; no newer agent build was found. Reviewed churn across all five clusters, benign startup-volume warnings and an expected Azure spot eviction. |
| 24 Sep, 06:18 UTC | Hot dev node and stage churn review | Dev node was reported at 90% memory at detection (2,513 MiB of 2,848 MiB); no MemoryPressure. Stage churn had no deployment shortfalls. Evidence: `incident-2026-09-24-hot-dev-node-and-stage-churn.md`. |
| 24 Sep, 08:40 UTC | Latest directly verified health check | 0 critical, 13 warnings, 10 info findings; `needsAttention=false`. The warnings were ongoing. Stage-cus overprovisioning was 1/2 available; customer impact was not confirmed. Cloudflare and Atlas remained unchecked. |
| 24 Sep, during this interview | Read-only release and support queries | Release status query showed app version 0.1.14 on dev, stage and prod, plus tenant `dan`; latest successful tenant deploy shown was 22 Sep. Support queue query returned 0 open tickets. These interview-time query results are not recorded in `log.md` or `support-log.md`. |

## Monitoring vs. production changes

Monitoring and investigation only; no production infrastructure changes, releases, tenant deployments or rollbacks were made. The ops log records four pending `consolidateAfter` 1m→10m approval cards; it notes one card delivery timed out. Live values remained 1m. Cloudflare and Atlas were not checked, so their health is unknown, not healthy.

## Current work and blockers

- `NoCompatibleInstanceTypes` warnings continue on the general NodePool across all five clusters. The controller-side cause remains a hypothesis because AKS-managed controller logs/config were unavailable. A shared chart fix needs a reviewed plan.
- Node consolidation caused transient platform-pod evictions. They recovered with no confirmed customer impact. Kyverno's single admission controller has no PDB, which creates risk during drains; no change was made.
- Monitoring-agent memory/OOM recurs fleet-wide. The object is AKS-managed, so a live patch would not be durable. A supported Microsoft tuning path or a decision to monitor/document is needed.
- Cloudflare and MongoDB Atlas checks are blocked by missing Vault credentials.
- Release automation is blocked for GitHub actions requiring the App credential: `cove-github-app` is missing from Vault. Read-only status did work.
- The available logs don't substantiate a 17-run count or any operations work on 22 Sep; the dated ops log starts on 23 Sep. The report therefore doesn't claim those as verified completed work.

## Next steps

1. Continue hourly AKS monitoring; investigate new critical/warning findings or a persistent deployment shortfall.
2. Decide on the shared Karpenter chart fix and safe apply path before any per-cluster change.
3. Confirm a supported tuning path for `ama-logs-rs`; avoid unsupported live patches.
4. Add Cloudflare, Atlas and GitHub App credentials to Vault if those checks/workflows are required.
5. Check scheduled daily-report and support-triage runs; their run counts were zero when schedules were inspected.
