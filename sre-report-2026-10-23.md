---
id: 5e1cfbc9-3f7d-4dd4-8727-0b995be1bbe3
binder: binder-mvvle32a-1
---
SRE report · 2026-10-23

# Infrastructure · Friday, 23 October 2026

**Verdict: Workloads available; stage telemetry OOM and dev metrics instability need attention. Cloudflare and Atlas not checked.**

Full health check with --diff at 08:00 EDT; read-only investigation through 08:03. Compared with 2026-10-22.md. No infrastructure or network changes made.

## At a glance

| Metric | Value | Detail | Status |
|---|---|---|---|
| Nodes ready | 31/31 | Initial snapshot; dev subsequently scaled from 7 to 5 nodes, all Ready | good |
| Unhealthy pods | 0 | Check definition; 617 pods, 206 deployments at desired replicas initially | good |
| Cloudflare 5xx | Not checked | Required Vault token absent | neutral |
| Atlas open alerts | Not checked | Required Vault service account absent | neutral |

Warnings rose from 7 yesterday to 8: 3 new, 5 ongoing, 2 cleared. No critical findings. This is not an end-to-end customer availability test. Dev metrics-server briefly fell to 1/2 available during investigation and returned to 2/2 by 08:03.

## Status

| System | Status | Key evidence |
|---|---|---|
| Prod eus2 | Available | 8/8 nodes Ready; yesterday's hot node now 79% memory |
| Prod cus | Available | 6/6 nodes Ready; 0 unhealthy pods |
| Stage eus2 | Watch | 5/5 nodes Ready; recovered telemetry OOM |
| Stage cus | Watch | 5/5 nodes Ready; small node at 106% allocatable memory |
| Dev | Watch | Initially 7/7, then 5/5 nodes Ready; metrics-server recovered to 2/2 |
| Cloudflare | Not checked | cloudflare Vault entry missing |
| MongoDB Atlas | Not checked | mongodb-atlas Vault entry missing |

## Needs a decision

- **Restore monitoring credentials.** Add cloudflare to Pennant Vault with the API token, including analytics-read access; add mongodb-atlas with service account client ID as username and client secret as secret. Certificate renewal status and database alerts remain unknown. No other secret stores were accessed.
- **Engineering review before changing managed agents or node sizing.** Validate supported telemetry memory settings and host memory accounting; investigate the AKS-managed metrics-server resource-resizing cycle. No restart or live patch is justified for the currently recovered agents. Any proposed production change requires founder approval.

## Watching

- **New stage eus2 telemetry OOM at 07:09 EDT.** Five kernel event records describe one cgroup OOM, not five separate crashes. ama-logs-mqp22 on system node vmss000001 terminated OOMKilled at the same time, against a 1Gi limit, and restarted once. It is Ready again. Container-limit exhaustion is confirmed; its growth trigger and any telemetry loss remain unverified. Review agent memory sizing rather than restarting the recovered pod.
- **New dev metrics-server readiness failures since at least 07:02 EDT.** Fourteen probe-failure events occurred in the check's last-hour window. Investigation caught 1/2 available; it recovered to 2/2 and the metrics API remained Available. Pod-nanny logs explicitly resize the deployment from 132Mi to 120Mi as node count changes. Repeated resizing during node churn is the leading explanation for rollout/probe noise, not a fully established root cause. Check reconciliation and sizing before modifying this AKS-managed deployment.
- **Memory headroom remains low.** New dev node aks-general-br8nr is at 95%; stage cus ddsh7 rose from 97% yesterday to 106% of allocatable memory. Both are Ready with MemoryPressure=False. Percentages above 100% refer to allocatable, not physical capacity. ama-logs uses 864Mi on dev and 958Mi on stage; summed container working sets are 2775Mi and 1916Mi versus node usage 5498Mi and 2960Mi. Host/system accounting needs investigation before attributing the entire gap to an application leak or approving resizing.
- **Ongoing stage cus capacity-match loop worsened.** NoCompatibleInstanceTypes events rose from 586 to 839 per hour. Desired deployment replicas were met; review NodePool requirements against regional capacity rather than changing the pool blindly.
- **Ongoing dev startup noise increased.** ama-logs FailedMount rose from 80 to 220, Defender FailedMount from 24 to 66, and overprovisioning FailedScheduling from 9 to 40. Total dev warning events rose from 799 to 1197. Monitor for persistent agent failures rather than suppressing these events.

## Resolved since yesterday

- Prod eus2 aks-general-mfrmm remains present and fell from 99% to 79% memory. This is recovery on the same node, not node replacement.
- Yesterday's prod eus2 OOM is absent from today's one-hour event window. The telemetry pod remains Ready; absence of a fresh event does not prove a permanent fix.
- Dev metrics-server recovered to 2/2 during this investigation without intervention. Recurring rollout instability remains under watch.

## Housekeeping

- Azure still advertises Kubernetes 1.37.0 for all five clusters; every inspected live node reports kubelet v1.36.4. Compatibility testing and approval must precede upgrades. Application versions were not checked.
- Certificate expiry is not freshly verified. The inventory's 30 September observation listed an earliest expiry of 5 November, now 13 days away. Restore Cloudflare access and verify renewal; do not treat that historical date as a current certificate check.
- Recurring image-policy and UnbalancedIRQs events remain informational. Engineering should confirm impact before cleanup or suppression.
- WARP was already manually disconnected when observed. All five APIs were reachable; its state was not changed.

Evidence: 2026-10-23-health.json and read-only kubectl snapshots in 2026-10-23-evidence/. Yesterday's comparison source: 2026-10-22.md.
