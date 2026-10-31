---
id: 15658f50-3c08-4949-b769-3e4b390a82f6
binder: binder-mvvle32a-1
---
SRE report · 2026-10-28

# Infrastructure · Wednesday, 28 October 2026

**Verdict: Workloads available; stage cus recovered from Spot preemptions. Cloudflare and Atlas not checked.**

Full health check with --diff at 08:00 EDT; read-only verification through 08:03 EDT. Compared with 27 October's local report and Binders note. No infrastructure or network settings changed. This is not an end-to-end customer availability test.

## At a glance

| Metric | Value | Detail | Status |
|---|---|---|---|
| Nodes ready | 29/29 | Final verification; 31/31 in the initial snapshot as stage cus nodes turned over | good |
| Unhealthy pods | 0 | Checker definition; 634 pods and all 224 deployments at desired replicas in the full check | good |
| Cloudflare 5xx | Not checked | Required Vault token absent | neutral |
| Atlas open alerts | Not checked | Required Vault service account absent | neutral |

## Status

| System | Status | Key evidence |
|---|---|---|
| Prod eus2 | Available | 8/8 nodes Ready; peak node memory 83%, up from 68% yesterday |
| Prod cus | Available | 6/6 nodes Ready; peak node memory 31% |
| Stage eus2 | Available | 6/6 nodes Ready; peak node memory 40%, down from 91% yesterday |
| Stage cus | Recovered, watching | 4/4 nodes Ready at final verification; logging, Defender and CNS agents all 4/4 Ready |
| Dev | Available, watching | 5/5 nodes Ready; peak node memory 72%; recurring startup warnings |
| Cloudflare | Not checked | No cloudflare entry in Pennant Vault; no current zone, tunnel or certificate verification |
| MongoDB Atlas | Not checked | No mongodb-atlas entry in Pennant Vault; no current cluster or alert verification |

## Needs a decision

- **Restore the monitoring credentials.** The existing board item remains open: add cloudflare with Zone Analytics Read, and mongodb-atlas with the service account client ID and secret. Yesterday's separate Key Vault checks are historical, not today's health evidence; no substitute credential route was used today.
- **Keep the telemetry resource review open.** Every cluster's ama-logs-rs pod has an OOMKilled last termination and is currently Ready. Both prod and stage clusters report one more lifetime restart than yesterday; counts can reset when pods are replaced. Engineering should validate supported persistent resource settings before any rollout. No live patch or restart is warranted by this snapshot.

## Watching

- **Three new stage cus warning groups recovered.** At 07:56 and 07:57 EDT, Azure recorded SpotEvictionIncoming for two nodes. Forced termination displaced add-ons; replacement nodes then logged CNI-not-initialized and object-not-registered mount errors. The full check counted 100 ama-logs mount warnings, 31 Defender publisher mount warnings and 15 overprovisioning scheduling warnings in its last-hour window. Scheduling events cite startup taints and insufficient reserved CPU. By final verification there were no Pending or crash-loop/image-pull pods, and logging, Defender and CNS agents matched all four remaining nodes. Later empty/underutilized consolidation also occurred. No persistent missing-volume failure was observed; no customer outage was established. Watch for sustained add-on unavailability before proposing a placement or capacity change.
- **Dev startup warnings continue.** 80 ama-logs mount, 25 Defender publisher mount and 16 overprovisioning scheduling events, all lower than yesterday's 180, 54 and 27. Current logging, Defender and CNS agents are 5/5 Ready; no node pressure conditions.
- **Telemetry OOMs remain a reliability concern.** Current ama-logs-rs lifetime restart counts are prod eus2 9, prod cus 4, stage eus2 7, stage cus 2 and dev 1; each has a 1536Mi memory limit. Ready agents do not prove successful downstream log ingestion, which was not tested.

## Resolved since yesterday

- Yesterday's hot dev and stage eus2 nodes are absent. Current peak memory is 72% and 40%, respectively, with no node pressure conditions anywhere. This is recovery through node turnover, not proof of a permanent capacity fix.
- The dev Defender eviction warning is absent from the current last-hour window; all collector and publisher DaemonSets are Ready. Existing placement review remains open.
- The full checker reports 0 critical, 6 warning and 17 informational findings: three new warnings, three ongoing and two prior warning keys cleared. Yesterday had five warnings. Upgrade-list changes are not incident resolutions.

## Housekeeping

- All observed kubelets and Azure control planes report 1.36.4. Azure now offers 1.36.5, 1.37.0 and 1.37.1 on all five clusters. Validate compatibility and schedule any approved upgrade one cluster at a time; no upgrade performed.
- Image-policy and UnbalancedIRQs events remain informational. Review their impact before cleanup or suppression.
- Certificate expiry was not checked today. The last verified earliest expiry was 12 December 2026, recorded yesterday; the inventory's older 5 November note is stale. Do not treat either historical observation as a fresh certificate check.

Evidence: 2026-10-28-health.json and reports/evidence/2026-10-28/ snapshots, including health.json and verification-summary.json. Comparison: 2026-10-27.md and Binders SRE report · 2026-10-27. Same report retained in Binders.
