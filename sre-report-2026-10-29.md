---
id: 37a5407a-1f48-48c5-a1e2-9704adb60b62
binder: binder-mvvle32a-1
---
SRE report · 2026-10-29
# Infrastructure · Thursday, 29 October 2026

**Verdict: Workloads available. Prod eus2 has one node at 96% memory with nobody under pressure yet, and the telemetry agents keep dying. Cloudflare and Atlas not checked.**

Full health check with --diff at 08:00 EDT, read-only verification through 08:05 EDT. Compared with the 28 October report. No infrastructure, workload or network settings changed. This is not an end-to-end customer availability test.

## At a glance

| Metric | Value | Detail | Status |
|---|---|---|---|
| Nodes ready | 29/29 | All five clusters, no pressure conditions anywhere | good |
| Unhealthy pods | 0 | Checker definition; 641 pods and all 224 deployments at desired replicas | good |
| Cloudflare 5xx | Not checked | Required Vault token absent | neutral |
| Atlas open alerts | Not checked | Required Vault service account absent | neutral |

## Status

| System | Status | Key evidence |
|---|---|---|
| Prod eus2 | Available, watching | 8/8 nodes Ready; node aks-general-4vdqv at 96% memory (2689Mi of 2781Mi allocatable); no MemoryPressure |
| Prod cus | Available | 6/6 nodes Ready; peak node memory 31% |
| Stage eus2 | Available | 6/6 nodes Ready; replacing a Spot node mid-morning, all agents back to 6/6 |
| Stage cus | Available, recovered | 4/4 nodes Ready; yesterday's mount, network and scheduling warnings all cleared |
| Dev | Available, watching | 5/5 nodes Ready; peak node memory 80%; 1197 warning events last hour, up from 820 |
| Cloudflare | Not checked | No cloudflare entry in Pennant Vault; no current zone, tunnel or certificate verification |
| MongoDB Atlas | Not checked | No mongodb-atlas entry in Pennant Vault; no current cluster or alert verification |

## Needs a decision

- **Prod eus2 node aks-general-4vdqv is at 96% memory.** The node is a 4 GB Standard_D2pls_v6 with 2781Mi allocatable and 2689Mi in use. It is not in pressure and nothing has been evicted, so no customer impact exists today. The cause is overcommit on a small node: pods there request 2438Mi (87%) and their limits add up to 13384Mi (481%). Most of the footprint is telemetry, an ama-logs agent at roughly 1005Mi and two otel-collector pods at 256Mi request and 1Gi limit each. My proposal, in order of risk: first spread the otel-collector replicas with an anti-affinity rule so the two do not land on the same node, then consider a larger node class for the general Karpenter pool. Confirm with Helder before either, since both touch production scheduling.
- **Restore the monitoring credentials.** Still open from the board: add a Vault entry `cloudflare` with an API token carrying Zone Analytics Read, and `mongodb-atlas` with the service account client ID and secret. Until then two of the seven systems in this report cannot be checked at all.

## Watching

- **The telemetry agents are dying and coming back all week.** ama-logs-rs restart counts are prod eus2 11 (last OOM 07:50 UTC today), stage eus2 8 (03:37), dev 3 (09:21), with prod cus and stage cus at 0. Alongside that, the per-node ama-logs DaemonSet has recorded OOM kills six times on prod eus2, three times on stage eus2 and twice on dev, most within the last 24 hours. On prod eus2 three of those agents sit at 933Mi to 1004Mi against a 1Gi limit, so they are at 91 to 98% of their ceiling and the next one is likely rather than unlikely. Restarts look like a 1536Mi limit on the replica set and 1Gi on the DaemonSet being too tight for current volume. Ready agents do not prove successful log ingestion, which is not tested here. Engineering should validate supported memory settings before any rollout; I am not patching a shared agent on a hunch.
- **Stage eus2 node turnover caused this morning's new mount warnings.** Karpenter removed node aks-general-g5jxn and launched aks-general-jqkpb 56 minutes before the check. The replacements logged 40 ama-logs-secret mount failures and 14 Defender secret mount failures plus CNI-not-initialized warnings while coming up. By verification every ama-logs pod was 2/2 Running, all six Defender publishers Running, and no pod was Pending. This is startup noise, not a storage fault. It will recur with every Spot replacement, and the durable fix is to raise the collector tolerations rather than chase the events.
- **Dev warning volume nearly doubled,** 1197 events against 820 yesterday, still the same three groups (mount, Defender publisher mount, overprovisioning scheduling). Logging and Defender agents are 5/5 Ready and peak node memory is 80%. Karpenter is churning nodes there as well.

## Resolved since yesterday

- All six stage cus warning groups are cleared: ama-logs and Defender publisher mount failures, overprovisioning scheduling failures, two NetworkNotReady groups and the Spot preemption notice. The two Spot nodes there were replaced and the cluster is now 4/4 Ready with no stray pods.
- Yesterday's hot stage eus2 node is gone; peak memory there is 42%.

## Housekeeping

- All five clusters run 1.36.4 and Azure offers 1.36.5, 1.37.0 and 1.37.1 on each. Validate compatibility and do any approved upgrade one cluster at a time, dev first. No upgrade attempted.
- UnbalancedIRQs events remain informational and noisy, 454 and 312 on the two prod eus2 system nodes, 303 on stage eus2, 46 and 42 on prod cus, 35 on stage cus. Worth a look before anyone suppresses them.
- Image-policy violations on verify-ghcr-images are steady at 333 on dev and 12 on prod eus2, informational.
- Certificate expiry was not checked today because the Cloudflare token is missing. The last verified earliest expiry was 12 December 2026, recorded on 28 October. Do not read that as a fresh check.

Evidence: 2026-10-29-health.json and reports/evidence/2026-10-29/, including the health snapshot and a verification summary. Comparison: 2026-10-28.md.
