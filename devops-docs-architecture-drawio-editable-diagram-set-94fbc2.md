---
id: b3da7332-2093-423b-9e72-5ef3a494fbc2
binder: binder-mvvle32a-1
---
DevOps | docs/architecture.drawio (editable diagram set)

Updated 2026-10-09 against origin/main (devplus-docs, docs/architecture.drawio)

`docs/architecture.drawio` is a source-XML, 5-page draw.io file covering the same architecture as `docs/DIAGRAMS.md`:

1. **System topology** — client surface → Cloudflare edge → tenant namespace (the 9 deployable resources) → job pods → hub services → external systems, plus the four key request flows (login, deploy, discover, chat/design).
2. **Edge ingress** — Tier 1 (Cloudflare anchored regex rules) vs Tier 2 (tenant nginx), with why anchoring and the ungated `/.well-known/` are load-bearing.
3. **Deployment lifecycle** — swimlane flow: POST → validate → deployment_queue → dispatcher lease → job pod (claim, creds, init+plan, estimate) → governance /evaluate → plan gate diamond → apply → cost_events → UNLOCK/ack.
4. **Queues & dispatcher** — the four Mongo queues, lease semantics, retry/crash behaviour, pod identity and tracing.
5. **Discovery** — trigger → scan job pod (buildProvider, creds, Scan(emit), upsert+gcOldRuns) → discovered_resources / discovery_results → consumption (resource query, EnvironmentFacts, agent tool, Architect agent).

**Conventions.** Draw.io's `image` style does NOT render embedded SVG in this app's export path; the icon chips put the SVG as a base64 `<img>` inside an HTML label instead. The generator script that emits the file lives outside the repo (temp) — regenerate by editing the XML directly, and keep the pages in sync with `docs/DIAGRAMS.md`, which remains the source of truth.
