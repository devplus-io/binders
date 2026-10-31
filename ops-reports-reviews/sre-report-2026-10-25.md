---
id: 0be0d543-e4f8-4324-afc3-8d0b117ebcb4
binder: binder-mvvle32a-1
folder: folder-mvw0u7ti
---
SRE report · 2026-10-25

# Infrastructure · Sunday, 25 October 2026

**Verdict: Workloads available; three new warning groups have recovered. Cloudflare and Atlas remain not checked.**

Full health check with --diff at 08:00 EDT, compared with 24 October's local report and Binders note. Targeted read-only checks followed at 08:02 to 08:04 EDT. No infrastructure or network settings changed. This is not an end-to-end customer availability test.

## At a glance

| Metric | Value | Detail | Status |
|---|---|---|---|
| Nodes ready | 27/27 | All five clusters; no node pressure conditions | good |
| Unhealthy pods | 0 | Down from 1 yesterday; 595 pods and all 206 deployments at desired replicas in the full check | good |
| Cloudflare 5xx | Not checked | Required Vault token absent | neutral |
| Atlas open alerts | Not checked | Required Vault service account absent | neutral |

Warnings remain at 11: 3 new, 8 ongoing, 3 previous warnings cleared. No critical findings. Node count fell from 28 to 27 through prod eus2 turnover; deployment availability did not fall.

## Status

| System | Status | Key evidence |
|---|---|---|
| Prod eus2 | Available | 7/7 nodes Ready; highest current node memory 67%, versus yesterday's hot node at 95% |
| Prod cus | Available | 6/6 nodes Ready; 0 unhealthy pods |
| Stage eus2 | Watch, recovered startup warnings | 5/5 nodes Ready; logging and Defender DaemonSets each 5/5 Ready |
| Stage cus | Watch, recovered controller warning | 4/4 nodes Ready; certificate controller 1/1 Ready |
| Dev | Available, recurring startup warnings | 5/5 nodes Ready; Defender collector now 5/5 Ready |
| Cloudflare | Not checked | cloudflare Vault entry missing |
| MongoDB Atlas | Not checked | mongodb-atlas Vault entry missing |

## Needs a decision

- **Restore monitoring credentials.** The existing board item remains open. Add cloudflare to Pennant Vault with analytics-read permission, and mongodb-atlas with its service account client ID and secret. Certificate renewal and database alerts remain unknown. No other secret store was accessed.
- **No immediate infrastructure change proposed.** Current agents and controller have recovered. Keep the existing engineering review of recurring telemetry/startup issues and dev system-node CPU reservations open; recovery alone does not establish a persistent fix.

## Watching

- **New stage cus certificate-controller readiness warning.** Seven HTTP 500 readiness failures were recorded from 04:59 to 07:27 EDT on an older pod. Events show subsequent underutilization evictions. Its replacement has been Ready since 07:35:45 EDT with zero restarts; current logs contain normal webhook-certificate updates and no errors in the sampled 100 lines. The original 500 cause is unconfirmed because the old pod is gone. No current controller outage is observed; watch for recurrence before proposing a change. No ExternalSecret resources were listed, so this does not demonstrate secret-refresh success.
- **New stage eus2 startup mount warnings.** The full check counted 40 ama-logs and 12 Defender publisher FailedMount events. Sample failures at 07:36 to 07:38 EDT say objects were not yet registered and coincide with new-node/pod startup. All current logging, Defender and CNS DaemonSets are 5/5 Ready. A transient startup registration delay is the leading explanation, not a proven permanent fault. No restart is warranted now.
- **Eight ongoing dev/stage cus warning groups.** Dev: ama-logs mounts 180, Defender publisher mounts 54, overprovisioning scheduling 35, CNS probes 9. Stage cus: 120, 38, 8 and 7 respectively. All current monitored agents are Ready. Counts are grouped event occurrences, not affected pod counts; retain the existing engineering review rather than suppressing warnings.

## Resolved since yesterday

- Dev Defender coverage recovered from 4/5 to 5/5. The system-node replacement collector has been Ready since 24 October 20:12:42 EDT and still requests 80m CPU across its two containers. No resource reduction was made by this run; the reason CPU headroom became available was not established.
- Prod eus2's hot node mfrmm is no longer present. Current node memory peaks at 67%. The alert cleared through node turnover, not a demonstrated memory fix on the same node.

## Housekeeping

- Azure advertises Kubernetes 1.37.0 for all five clusters; every node checked reports kubelet v1.36.4. Engineering should validate compatibility before seeking upgrade approval. Application versions were not checked.
- Certificate expiry remains unverified. The inventory's 30 September observation listed an earliest expiry of 5 November, now 11 days away. This is historical evidence only; restore access and verify renewal promptly.
- Image-policy, UnbalancedIRQs and preemption events remain informational. Review impact before any cleanup or suppression.

Evidence: evidence/2026-10-25/health.json, verification-summary.json and timestamped read-only node, pod, DaemonSet, event, metrics and controller snapshots. Comparison: 2026-10-24.md and Binders SRE report · 2026-10-24. Report also retained in Binders.
