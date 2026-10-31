---
id: 2dc9a87b-7118-4dfc-b118-a9c1af2dd039
binder: binder-mvvle32a-1
folder: folder-mvw0u7ti
---
SRE report · 2026-10-26

# Infrastructure · Monday, 26 October 2026

**Verdict: Monitoring degraded: all five AKS workload checks are blocked by this Mac's private-network path. Customer availability is unverified.**

Full health check with --diff at 08:00 EDT; read-only DNS, routing and API verification through 08:03 EDT. Compared with 25 October's local report and Binders note. No infrastructure, VPN or DNS settings changed; WARP stayed Connected.

## At a glance

| Metric | Value | Detail | Status |
|---|---|---|---|
| Nodes ready | Not checked | 0/5 cluster APIs reachable; yesterday 27/27 nodes Ready | neutral |
| Unhealthy pods | Not checked | Yesterday 0; today's API timeouts prevent a count | neutral |
| Cloudflare 5xx | Not checked | Required Vault token absent | neutral |
| Atlas open alerts | Not checked | Required Vault service account absent | neutral |

The checker returned five new critical API-access findings, zero warnings and five upgrade notices. Zero warnings does not mean yesterday's issues recovered: the workload checks could not run. Azure management reports all five clusters Running with provisioning Succeeded, which does not verify workload health.

## Status

| System | Status | Key evidence |
|---|---|---|
| Prod eus2 | Not checked | API timeout; Azure Running/Succeeded |
| Prod cus | Not checked | API timeout; Azure Running/Succeeded |
| Stage eus2 | Not checked | API timeout; Azure Running/Succeeded |
| Stage cus | Not checked | API timeout; Azure Running/Succeeded |
| Dev | Not checked | API timeout; Azure Running/Succeeded |
| Cloudflare | Not checked | cloudflare Vault entry missing |
| MongoDB Atlas | Not checked | mongodb-atlas Vault entry missing |

## Needs a decision

- **Review the Mac's overlapping VPN DNS and routes.** A new infrastructure to-do records the evidence and proposed fix: stop Tailscale from accepting DNS and subnet routes on this Mac, or narrow the conflicting routes, while leaving WARP Connected. Check Helder's other connectivity needs before changing it. After the change, rerun the full check and verify all five APIs and workloads. No network change was applied.
- **Restore monitoring credentials.** The existing to-do remains open: add cloudflare with analytics-read permission and mongodb-atlas with its service account client ID and secret. Certificate renewal and database alerts remain unknown. No other secret store was accessed.

## Watching

- **New shared AKS access failure, first observed at 08:00 EDT.** All five cluster API requests timed out in the full check and bounded follow-up. This blocks node, pod and deployment inspection across prod, stage and dev. It is not evidence that all five clusters or customer applications are down; the actual onset is unknown.
- **DNS conflict confirmed; route conflict is the leading connectivity explanation.** macOS installs supplemental AKS private-domain resolvers on Tailscale's utun0. Queries to its 100.100.100.100 resolver return SERVFAIL for the sampled dev endpoint, while WARP's 127.0.2.2 resolver and Azure DNS return 10.2.0.10. WARP resolves all five private API names. All five API IP routes nevertheless select utun0, and direct TCP 443 probes time out; bypassing hostname resolution with TLS hostname verification retained also times out. WARP reports Connected and Network healthy. This rules out WARP being disconnected and rules out a DNS-only explanation; restoring route selection and repeating the checks would confirm the remaining cause. No evidence of an Azure cluster power/provisioning failure was found.
- **Yesterday's eleven warning groups are unverified today.** The dev/stage logging, Defender, scheduling and CNS warnings and stage cus certificate-controller warning cannot be reassessed. Keep the existing engineering reviews open; yesterday's recovered agents do not prove today's health.

## Resolved since yesterday

- No workload resolution can be verified. The --diff output labels missing previous findings resolved because today's cluster checks stopped at API access. Those are monitoring gaps, not demonstrated recoveries.
- Compared with yesterday, coverage fell from five reachable clusters with 27 Ready nodes and 206 available deployments to no workload visibility. The Cloudflare and Atlas credential gaps are unchanged.

## Housekeeping

- Azure now advertises Kubernetes 1.36.5 and 1.37.1 as well as 1.37.0 for both centralus clusters; yesterday's report listed only 1.37.0. Eastus2 clusters still advertise 1.37.0. Azure reports control-plane version 1.36.4; kubelet and application versions were not checked today. Validate compatibility before seeking upgrade approval.
- Certificate expiry remains unverified. The inventory's 30 September observation listed an earliest expiry of 5 November, ten days away. This historical observation is not confirmation of the current certificate or renewal status; verify promptly after restoring access.
- Existing telemetry/startup and dev system-node capacity reviews remain open. No restart, rollout, upgrade or warning suppression is justified by today's missing workload data.

Evidence: evidence/2026-10-26/health.json, api-verification.json, dns-warp.txt, dns-isolation.json, warp-api-verification.json and private-route-verification.json. Comparison: 2026-10-25.md and Binders SRE report · 2026-10-25. This report is also retained in Binders.
