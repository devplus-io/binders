---
id: e82eaa6a-46be-46cb-bfa3-4f7d0f811610
binder: binder-mvvle32a-1
folder: folder-mvw0u7ti
---
SRE report · 2026-10-24

# Infrastructure · Saturday, 24 October 2026

**Verdict: Workloads available in the morning check; dev Defender coverage is incomplete and prod eus2 memory needs attention. Cloudflare and Atlas not checked.**

Full health check with --diff at 08:23 EDT; compared with 23 October's local report and Binders note. Final targeted verification at 11:40 EDT. Read-only throughout; no infrastructure or network settings changed.

## At a glance

| Metric | Value | Detail | Status |
|---|---|---|---|
| Nodes ready | 28/28 | Full check and final API recheck | good |
| Unhealthy pods | 1 | Dev Defender collector still Pending at final verification | watch |
| Cloudflare 5xx | Not checked | Required Vault token absent | neutral |
| Atlas open alerts | Not checked | Required Vault service account absent | neutral |

Morning snapshot: 609 pods; all 206 deployments at desired replicas. No critical findings. Warnings increased from 8 yesterday to 11: 8 new, 3 ongoing, 5 previous warnings absent. The diff also cleared 3 informational findings. This is not an end-to-end customer availability test.

## Status

| System | Status | Key evidence |
|---|---|---|
| Prod eus2 | Watch | 8/8 nodes Ready; aks-general-mfrmm memory 94% initially, 95% at final check |
| Prod cus | Available in full check | 6/6 nodes Ready; 0 unhealthy pods in morning snapshot |
| Stage eus2 | Available in full check | 5/5 nodes Ready; no fresh OOM warning in the check window |
| Stage cus | Watch | 4/4 nodes Ready; startup warnings, but all current network and monitoring agents Ready at final check |
| Dev | Degraded agent coverage | 5/5 nodes Ready; Defender collector DaemonSet 4/5 Ready at final check |
| Cloudflare | Not checked | cloudflare Vault entry missing |
| MongoDB Atlas | Not checked | mongodb-atlas Vault entry missing |

All five APIs answered the final node check. Earlier follow-up requests timed out across all five clusters; the bounded retry also failed, then access recovered without intervention. Workloads were not observable during that gap. This establishes intermittent monitoring reachability from this Mac, not a customer outage or a confirmed network cause.

## Needs a decision

- **Restore monitoring credentials.** The existing board item remains open: add cloudflare to Pennant Vault with analytics-read permission, and mongodb-atlas with its service account client ID and secret. Database alerts and certificate renewal remain unknown. No alternative secret stores were accessed.
- **Engineering review of dev system-node CPU reservations.** The Defender pod is restricted to the system node. Its node description confirms 1826m of 1900m CPU already requested, leaving 74m for a pod requesting 80m. Have engineering propose a persistent placement change for eligible controllers or a supported system-pool capacity change. Do not reduce security-agent requests merely to make it fit. Any resize or rollout needs approval; no change was made.

## Watching

- **New dev Defender coverage gap since 04:12 EDT.** microsoft-defender-collector-ds-pbb7s remained Pending through 11:40, over seven hours. The full check recorded 16 scheduling warnings. Four other nodes are excluded by the DaemonSet's node affinity. The eligible system node has only 74m CPU unreserved, less than the pod's 80m request; this confirms the scheduling constraint despite low actual CPU use. The system node lacks its low-level collector; the publisher remains Ready. Restarting the pod would not free the required CPU reservations.
- **Prod eus2 memory regression.** aks-general-mfrmm rose from 79% yesterday to 94% in the full check and 95% at final verification. In the morning snapshot it was Ready with MemoryPressure=False. Its telemetry agent used 575Mi, with other pods and host memory also contributing. No leak or single offending workload is established. Continue the existing engineering review of agent sizing and host accounting; any production change needs founder approval.
- **New stage cus startup warnings.** The check counted 120 ama-logs FailedMount events, 39 Defender publisher FailedMount events, 10 overprovisioning scheduling failures and 5 azure-cns probe failures. Sample events show objects not yet registered and CNS connection refusals during startup. All current CNS, ama-logs and Defender agents were Ready at final verification, and no pods were Pending. Node churn is a leading explanation for this startup pattern, not a proven permanent fault. Do not restart recovered agents or suppress the warnings.
- **Dev startup warnings continue.** ama-logs FailedMount fell from 220 to 160, Defender publisher FailedMount from 66 to 48, and overprovisioning scheduling failures from 40 to 25. New CNS probe failures totalled 8. All current CNS, ama-logs and Defender publisher pods were Ready at final verification. Keep watching for persistent failures rather than treating recovered startup events as an outage.
- **Monitoring access was intermittent.** WARP reported manually Disconnected when checked, as yesterday. Its settings were left untouched. Access recovered without a WARP change; the cause of the temporary API timeouts remains unverified.

## Resolved since yesterday

- Yesterday's hot dev node br8nr and stage cus node ddsh7 are no longer present. Their warnings cleared with node turnover, not demonstrated recovery of the same nodes.
- Stage cus NoCompatibleInstanceTypes warnings are absent from the current check window. That does not prove the regional capacity constraint is permanently fixed.
- Yesterday's stage eus2 OOM warning is absent from the current check window; its telemetry agents were Ready in the morning snapshot. No permanent memory fix was applied.
- Dev metrics-server readiness warnings cleared. Both replicas were Ready in the morning snapshot and at final verification.

## Housekeeping

- Azure advertises Kubernetes 1.37.0 for all five clusters. Every node inspected in the morning snapshot reported kubelet v1.36.4. Engineering should validate compatibility and obtain approval before upgrades; application versions were not checked.
- Certificate expiry is not freshly verified. The inventory's 30 September observation listed an earliest expiry of 5 November, now 12 days away. Treat that as historical evidence only; restore access and check renewal promptly.
- Image-policy and UnbalancedIRQs events remain informational. Engineering should confirm impact before cleanup or suppression.
- The existing telemetry and metrics-server investigation remains open. Add today's CPU-reservation evidence to the review; no infrastructure mutations were made.

Evidence: 2026-10-24-health.json and read-only snapshots in 2026-10-24-evidence/. Comparison sources: 2026-10-23.md and Binders SRE report · 2026-10-23. This report is also saved in Binders.
