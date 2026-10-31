---
id: d383b25d-da31-4cdb-9c25-6cb2c7ddb04d
binder: binder-mvvle32a-1
folder: folder-mvw0u7ti
---
SRE report · 2026-10-20
# Infrastructure · Tuesday, 20 October 2026

**Verdict: Monitoring blocked.** All five private AKS APIs are unreachable from this Mac. Azure reports all five clusters Running/Succeeded, but workloads could not be checked. A customer-facing outage is not established. Cloudflare and MongoDB Atlas also remain unchecked because the required Vault entries are missing.

Full health check with `--diff` completed at 19:47 UTC. It produced five critical API-reachability findings and five informational upgrade findings. These are monitoring findings, not proof that five clusters are down.

## At a glance
| Metric | Value | Status |
|---|---|---|
| Nodes ready | Not checked | neutral |
| Unhealthy pods | Not checked | neutral |
| Cloudflare 5xx rate | Not checked | neutral |
| Atlas open alerts | Not checked | neutral |

## Status
| System | Status | The number that matters |
|---|---|---|
| Prod eus2 | Not checked | API timeout; Azure Running/Succeeded |
| Prod cus | Not checked | API timeout; Azure Running/Succeeded |
| Stage eus2 | Not checked | API timeout; Azure Running/Succeeded |
| Stage cus | Not checked | API timeout; Azure Running/Succeeded |
| Dev | Not checked | API timeout; Azure Running/Succeeded |
| Cloudflare | Not checked | Missing `cloudflare` Vault entry |
| MongoDB Atlas | Not checked | Missing `mongodb-atlas` Vault entry |

## Needs a decision
- **Confirm the intended Covee access configuration for this Mac.** Your 19 October Binders note says Covee is replacing Warp. Before changing DNS or network settings, confirm how this Mac should reach private AKS endpoints through the replacement. No cloud or DNS configuration was changed.
- **Provide the two read-only service credentials through Pennant's Vault.** The check requires `cloudflare` and `mongodb-atlas`. Until available, 5xx rates, tunnel health, certificate expiry and database alerts cannot be verified by this check.

## Watching
- **Private access failure observed today.** Warp was manually disconnected when this task resumed. I reconnected it under the standing keep-connected rule; it now reports Connected/healthy, but private access still fails. macOS routes the AKS private DNS zones to Tailscale's 100.100.100.100, which times out; Tailscale reports stopped. Querying Warp's local resolver directly resolves the prod-eus2 API to 10.20.0.10, but TCP 443 and a certificate-name-preserving Kubernetes request to that IP also time out. DNS alone therefore does not explain the whole failure. Route inspection selects utun0. A stale or conflicting private-network configuration is the leading hypothesis; validating the intended Covee DNS and route setup would confirm the fix. The exact failure start and customer impact are unknown.
- **Comparison baseline is incomplete.** No 19 October daily report was found in the reports folder or the DevPlus Binders searches and recent notes. The latest saved local report is 15 October: 28/28 nodes ready, no unhealthy pods. A separate 16 October check from this interrupted task also reached all five clusters, but is historical evidence only. Today's change is loss of monitoring access, not a verified workload regression.

## Resolved since yesterday
None can be verified. The script marks earlier pod and event findings as resolved because they are absent from today's results, but those sources were unreachable. Prior memory, monitoring-agent restart and scheduling warnings remain unverified, not resolved.

## Housekeeping
Azure now reports control-plane version 1.36.4 and offers 1.37.0 on all five clusters, compared with 1.35.8 in the 15 October report. Node and workload versions could not be verified. Do not schedule another upgrade based on this incomplete check. Certificate expiry and current event noise were not checked.

No production resources were changed, restarted or deleted. The only local change was reconnecting Warp; it remains Connected. This report is saved locally and in the DevPlus Binders knowledge base. Evidence: `/tmp/health-2026-10-20.json`, plus read-only DNS, route and TCP probes. The task resumed on 20 October after an inference/network interruption; 16 October observations are not presented as today's health.
