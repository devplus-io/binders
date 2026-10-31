---
id: a582e7aa-451a-4d2a-8915-67f7eebdf3b0
binder: binder-mvvle32a-1
folder: folder-mvw0u7cf
---
Security | internal-token auth hardening (2026-10-14 audit, landed 2026-10-23)

Updated 2026-10-23 against local commits in devplus-backend, devplus-oidc-service, devplus-opentofu-server, devplus-executor, devplus-platform, devplus-frontend, docs and devplus-core-infra (committed locally, not yet pushed).

The 2026-10-14 audit's auth findings are implemented in code. What is true now:

- **oidc-service** /token/project and /token/deployment require X-Internal-Token (or Bearer) equal to INTERNAL_API_TOKEN, constant-time, 503 when unconfigured. JWT_SECRET is no longer consumed: validation is RS256-only and the HMAC fallback whose default was the repo-literal "default-secret-change-in-production" is gone. JWT_SECRET stays deployed for chart compatibility only.
- **opentofu-server** /state/* (GET/POST/DELETE/LOCK/UNLOCK) and registry upload/delete require the same token; registry reads (versions, download, module.tar.gz) stay open because the tofu registry protocol carries no secrets. tofu gets the header through backend.hcl headers (executor opentofu_executor.go); StateClient, RegistryPusher and the OIDC mint calls stamp it. INTERNAL_API_TOKEN is injected from the chart's internalApi.tokenSecret.
- **backend**: the by-id deployment handlers now enforce deployment.ProjectID == the path {projectId} (404 on mismatch, no existence disclosure) — H1 IDOR; a token with no aud claim is rejected (M2).
- **Chart**: OIDC_ALLOWED_SERVICE_ACCOUNTS must include <namespace>-app whenever serviceAccount.create is true, because job pods run as that SA and mint tokens via /token/service for live log broadcast. Commit 8a599d5 added it; an uncommitted edit had silently reverted it and was restored on 2026-10-23.
- **NetworkPolicy** networkPolicy.opentofuServer is opt-in and defaults to off: backend, dispatcher and deployment/module/adoption job pods may reach :8082. It is defence in depth, not a trust boundary — job pods run user module code and can still read every deployment's state through that port.
- **Frontend** nginx sends HSTS, X-Content-Type-Options and Referrer-Policy (pentest F4).
- **core-infra**: the prod workspace owns the devplus.io zone's single http_request_firewall_custom ruleset (WorkOS webhook IP allowlist, unused-method block, Tor challenge).
- **scripts/rotate-tilt-secrets.sh** creates/rotates the per-tenant tilt-jwt-<tenant> and tilt-kek-<tenant> Key Vault secrets, keeping prior KEKs for multi-version unsealing.

The internal token is namespace-wide: it stops any pod in the cluster reaching these services, not a compromised peer inside the tenant.

Deploy-path change in the same batch: the canary quality-gate job and the quality_gate input were removed from _deploy-environment.yml, deploy-tenants.yml and release.yml. AGENTS.md §12 still describes canary-then-gate promotion; reconcile before relying on the gate.
