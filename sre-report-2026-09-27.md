---
id: e54d7cc5-7a24-4cbc-9856-2af3de40dca1
binder: binder-mvvle32a-1
---
SRE report · 2026-09-27

# Infrastructure · Sunday, 27 September 2026

**Watch — AKS is serving normally across five clusters, but the autoscaling warning remains and Cloudflare and Atlas are not checked.** Full `health.py --diff` completed at 12:00 UTC. It found 0 critical, 8 warnings (2 newly flagged since the previous health run), and 12 informational findings. All 27/27 nodes are Ready; no unhealthy pods or deployments short of replicas. This was a read-only check; no infrastructure changes were made.

## At a glance

| Measure | Today | Yesterday (26 Sep) |
|---|---:|---:|
| Nodes ready | 27/27 | 25/25 |
| Unhealthy pods / short deployments | 0 / 0 | 0 / 0 |
| Warnings | 8, including 2 new transient startup events | 11 |
| Cloudflare 5xx and Atlas open alerts | Not checked | Not checked |

## Status

| System | Status | Number that matters |
|---|---|---|
| Prod eus2 | Running; workload checks good; autoscaling warning | 6/6 Ready, 0 unhealthy pods, 0 short deployments; AKS and all inspected nodes v1.35.8 |
| Prod cus | Running; workload checks good; autoscaling warning | 6/6 Ready, 0 unhealthy pods, 0 short deployments; AKS v1.35.7 |
| Stage eus2 | Running; workload checks good; autoscaling warning | 5/5 Ready (4 yesterday), 0 unhealthy pods, 0 short deployments; AKS and all inspected nodes v1.35.8 |
| Stage cus | Running; workload checks good; autoscaling warning | 4/4 Ready, 0 unhealthy pods, 0 short deployments; AKS v1.35.7 |
| Dev eus2 | Running; workload checks good; memory and autoscaling warnings | 6/6 Ready (5 yesterday), 0 unhealthy pods, 0 short deployments; AKS and all inspected nodes v1.35.8 |
| Cloudflare | **Not checked** | Missing Vault entry `cloudflare`: 5xx, tunnels, zones and certificate expiry unknown |
| MongoDB Atlas | **Not checked** | Missing Vault entry `mongodb-atlas`: clusters and alerts unknown |

## Needs a decision

- **Restore coverage:** Add the `cloudflare` API token and `mongodb-atlas` service-account credentials in Pennant's Vault. No credential values belong in this report. Without them two systems remain unchecked.
- **Autoscaling risk (ongoing):** `NodePool/general` reports `NoCompatibleInstanceTypes` in every cluster (52–552 occurrences per cluster in the last hour). This does not show a current workload outage, but provisioning for a burst could fail. Investigate the NodePool/NodeClass requirements and prepare an environment-specific change for approval; do not live-patch or alter production without a separate approved card and Founders' review where applicable.

## Watching

- **New dev startup warnings, now recovered:** At 11:29:17 UTC the node `aks-ondemand-p9wcg` registered. From 11:29:19–11:29:27 UTC its new `ama-logs-nbvm2` and `microsoft-defender-publisher-ds-68shz` pods had 20 and 5 `FailedMount` events plus 5 and 6 `NetworkNotReady` events, respectively. Messages said Kubernetes objects had not registered yet and CNI had not initialized. By 11:30:05 both pods had started; read-only inspection at report time found them Running/Ready with zero restarts, and the node Ready. This supports a brief node-startup race, not a persistent missing-secret incident. The check still counted the events within its last-hour window. Watch recurrence or prolonged unready pods; no change proposed now.
- **Dev node memory:** `aks-general-l2pzj` is at 101% of allocatable, versus two different nodes at 91–92% yesterday. Node inspection showed Ready and `MemoryPressure=False`; continue watching for pressure or evictions. No causal claim about the workload without further inspection.
- **Event trends:** 8 warnings versus 11 yesterday. Stage-cus `ama-logs` and Defender `FailedMount`, overprovisioning `FailedScheduling`, and the network-not-ready events cleared from the diff. Stage-cus warning events fell 604→244/h; dev fell 1900→1349/h. Stage-eus2 rose 378→554/h and prod-cus rose 51→137/h, mainly to watch alongside autoscaling, rather than claiming an across-the-board improvement.
- Dev `verify-ghcr-images` PolicyViolation remains informational (378 events/h); some `UnbalancedIRQs` informational events persist.

## Resolved since yesterday

- Yesterday's two dev nodes above the memory watch threshold (`aks-ondemand-mzs25`, `aks-ondemand-vkjcq`) are no longer flagged; a different dev node now exceeds the threshold. This is **not** resolution of memory pressure in dev overall.
- Stage-cus mount, network and scheduling warning groups listed above are no longer present in the diff. The diff also marked an `external-secrets-cert-controller` Unhealthy event and one prod-eus2 `UnbalancedIRQs` node event resolved since an intervening check; those were not in yesterday's daily report.
- All five clusters still have zero unhealthy pods and full deployment replicas; stage-eus2 and dev each have one more Ready node than yesterday.

## Housekeeping

- AKS control-plane and inspected node versions changed from v1.35.7 yesterday to v1.35.8 in dev, stage eus2, and prod eus2. This report **observes** the version change; it does not attribute or claim authorization for the upgrade. Prod cus and stage cus remain v1.35.7. Available upgrade lists include v1.36.x; the check lists no v1.35.8 patch offer for the two central-US clusters. No upgrade was performed by this report.
- Cloudflare certificate expiry was not checked due to missing access.
- Compared against `2026-09-26.md`; the previous-health-run diff is not itself a day-over-day diff. No infrastructure changes were made in this task.
