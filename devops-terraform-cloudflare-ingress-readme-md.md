---
id: 8354dda6-922c-4dbf-837b-51b266ccb6b4
binder: binder-mvvle32a-1
---
DevOps | terraform-cloudflare-ingress/README.md

Source: DevPlusInc/terraform-cloudflare-ingress/README.md, branch docs/devops-refresh-2026-10, refreshed 2026-10-04

Refreshed 2026-10-04 against origin/main code (DevPlusInc/terraform-cloudflare-ingress, README.md).

# Cloudflare module

Single module for the full Cloudflare tunnel setup: one Zero Trust Tunnel per namespace, tunnel config, DNS CNAME, and cloudflared Deployment in that namespace.

## What it creates

- **Cloudflare:** Tunnel `<tunnel_name_prefix>-<subdomain>` (remotely managed, `config_src = "cloudflare"`), its ingress config, and a proxied CNAME `<subdomain>.<base_domain>` → `<tunnel-id>.cfargotunnel.com`. `subdomain` defaults to `namespace`.
- **Kubernetes:** Secret `cloudflared-tunnel-token` (key `token`) and Deployment `cloudflared` (1 replica) running `cloudflared tunnel run --token $(TOKEN)`.

Ingress rules for the hostname, first match wins. The issuer-root and MCP paths are anchored regexes, because cloudflared matches `path` unanchored and `/api/v1/...` forms of them must still reach the backend:

| Path | Service |
|------|---------|
| `^/.well-known/openid-configuration$`, `^/.well-known/jwks.json$` | `http://oidc-service:8081` (OIDC discovery + JWKS at the issuer root) |
| `^/.well-known/oauth-protected-resource$`, `^/.well-known/oauth-protected-resource/mcp$`, `^/mcp/?$` | `http://devplus-mcp:8085` (hosted MCP service) |
| `/api/v1/.*` | `http://backend:8080` |
| anything else on the hostname | `http://frontend:80` |
| catch-all | `http_status:404` |

The namespace must already exist (created by `terraform-helm-devplus-services`). Cloudflared runs in the same namespace as the app, so the tunnel config uses in-namespace Service names.

## Usage

```hcl
module "cloudflare-ingress" {
  source = "github.com/DevPlusInc/terraform-cloudflare-ingress?ref=<commit>"

  providers = {
    cloudflare = cloudflare
  }
  account_id  = var.cloudflare_account_id
  zone_id     = var.cloudflare_zone_id
  namespace   = module.helm-deployment.namespace
  subdomain   = "tftest.dev"  # optional: hostname tftest.dev.devplus.io instead of <namespace>.devplus.io
  base_domain = "devplus.io"  # optional, default

  depends_on = [module.helm-deployment]
}
```

`terraform-devplus-tenant` is the caller: it passes `subdomain = "<namespace>.<environment>"` (bare `<namespace>` in prod), so this hostname matches the tenant's public host and OIDC issuer.

## Inputs

| Variable | Default | Description |
|----------|---------|-------------|
| `account_id` | (required) | Cloudflare account ID |
| `zone_id` | (required) | Zone ID for `base_domain` |
| `namespace` | (required) | Namespace where the app and cloudflared run |
| `base_domain` | `devplus.io` | Base domain for the hostname |
| `subdomain` | null | DNS name under `base_domain`; null uses `namespace` |
| `tunnel_name_prefix` | `devplus` | Tunnel name = `<prefix>-<subdomain>` |
| `cloudflared_image` | `cloudflare/cloudflared:latest` | cloudflared image |

## Outputs

| Output | Value |
|--------|-------|
| `tunnel_id` | Cloudflare tunnel ID |
| `hostname` | Public hostname (`<subdomain>.<base_domain>`) |
| `cname_target` | `<tunnel-id>.cfargotunnel.com` |

## Requirements

- Cloudflare API token with Tunnel Write and DNS edit for the zone.
- Provider `cloudflare/cloudflare ~> 5.17.0`. The Kubernetes resources use the caller's default `hashicorp/kubernetes` provider (not declared here).
- Kubernetes namespace created before applying (e.g. `depends_on = [module.helm-deployment]`).
- Services `frontend`, `backend` and `oidc-service` in that namespace (from the DevPlus Helm chart), plus `devplus-mcp` when the chart runs it with `devplusMcp.enabled`.
- No CI workflow in this repository.
