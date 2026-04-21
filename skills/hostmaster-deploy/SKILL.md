> **Part of [Consola Skills](https://github.com/oberon-logistics/consola-skills) — battle-tested Hermes agent skills**

---
name: hostmaster-service-deploy
version: 1.0.0
description: >
  Deploy a new Docker service behind the Cloudflare tunnel → Caddy → Authentik SSO
  stack on the hostmaster infrastructure. Covers compose config, Caddy routing,
  DNS/tunnel ingress, and SSO protection.
metadata:
  hermes:
    tags: [docker, deploy, cloudflare, caddy, authentik, traccar, hostmaster, self-hosted]
    triggers: [deploy service, add service, new container, hostmaster]
    requires_toolsets: [terminal, file]
---

# Hostmaster Service Deployment

Pattern for adding a new Docker service to the existing self-hosted infrastructure
on `thor`. The stack uses Cloudflare tunnel → Caddy reverse proxy → backend services,
with Authentik SSO for protected endpoints.

## Architecture Overview

```
Internet → Cloudflare (DNS + Tunnel) → Caddy (reverse proxy) → Service containers
                                         ↓
                                    Authentik (forward_auth for SSO)
```

## Infrastructure Layout

| Path | Purpose |
|------|---------|
| `~/hostmaster/platform/` | Core stack: Caddy, Cloudflared, Forgejo, Authentik, Postgres, Redis |
| `~/hostmaster/platform/docker-compose.yml` | Platform services compose |
| `~/hostmaster/platform/caddy/Caddyfile` | Caddy reverse proxy config |
| `~/hostmaster/platform/cloudflared/config.yml` | Tunnel config (routing via API) |
| `~/hostmaster/internal/` | Internal services (Memos, Traccar, etc.) |
| `~/hostmaster/internal/docker-compose.yml` | Internal services compose |
| `~/hostmaster/scripts/cf-add-hostname.sh` | Add DNS CNAME + tunnel ingress in one step |

## Docker Networks

| Network | Purpose | Who's on it |
|---------|---------|-------------|
| `external-net` | Cloudflared ↔ Caddy | cloudflared, caddy |
| `platform-net` | Caddy ↔ backend services | caddy, forgejo, authentik, postgres, redis, **new services** |
| `sites-net` | Caddy ↔ public websites | caddy, blog containers |
| `internal-net` | Internal-only services | memos (not proxied through Caddy) |

**Key rule:** Any service that Caddy reverse-proxies to MUST be on `platform-net`.

## Step-by-Step Deployment

### 1. Add service to compose file

For platform services, edit `~/hostmaster/platform/docker-compose.yml`.
For internal services, edit `~/hostmaster/internal/docker-compose.yml`.

If adding to internal compose and the service needs Caddy access, add `platform-net` as an external network:

```yaml
services:
  newservice:
    image: newservice/image:latest
    restart: unless-stopped
    volumes:
      - newservice_data:/data
      - ./newservice/config:/config:ro
    networks:
      - platform-net

volumes:
  newservice_data:

networks:
  internal-net:
    name: internal-net
  platform-net:
    external: true
```

**Do NOT expose ports with `ports:` unless the service also needs direct Tailscale access.**
Caddy reaches services by container name on the Docker network — no port mapping needed.

### 2. Add Caddy route

Edit `~/hostmaster/platform/caddy/Caddyfile`.

**Public service (no auth):**
```
http://newservice.yourdomain.org {
    reverse_proxy newservice:8080
}
```

**With native OIDC (app handles its own auth, e.g., Dawarich):**
Same as above — simple reverse_proxy. Do NOT use forward_auth.

**SSO-protected service (Caddy forward_auth):**
```
http://newservice.yourdomain.org {
    forward_auth authentik-server:9000 {
        uri /outpost.goauthentik.io/auth/caddy
        copy_headers X-authentik-username X-authentik-groups X-authentik-email
    }
    reverse_proxy newservice:8080
}
```

**Note:** TLS is terminated by Cloudflare. Caddy runs plain HTTP (`auto_https off`).
Use `http://` prefixes, not `https://`.

### 3. Add DNS + tunnel ingress

```bash
cd ~/hostmaster
set -a && source platform/.env && set +a
bash scripts/cf-add-hostname.sh newservice.yourdomain.org
```

This creates the Cloudflare DNS CNAME record AND adds the hostname to the tunnel
ingress config in one step. The tunnel routes all hostnames to Caddy, which then
routes by hostname to the correct backend.

### 4. Start the service

```bash
# If added to internal compose:
cd ~/hostmaster/internal && docker compose up -d

# If added to platform compose:
cd ~/hostmaster/platform && docker compose up -d
```

### 5. Verify

```bash
# Container is running
docker ps --filter name=newservice

# Public URL responds
curl -sI https://newservice.yourdomain.org
```

### 6. Reload Caddy (if needed)

If Caddy doesn't pick up the new route automatically:
```bash
docker exec platform-caddy-1 caddy reload --config /etc/caddy/Caddyfile
```

## Pitfalls

- **Java Properties XML:** Some services (like Traccar) use Java's Properties XML format
  which REQUIRES the DOCTYPE declaration:
  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <!DOCTYPE properties SYSTEM 'http://java.sun.com/dtd/properties.dtd'>
  <properties>
      <entry key='key'>value</entry>
  </properties>
  ```
  Missing DOCTYPE causes `SAXParseException` and container restart loops.

- **Container must be on `platform-net`** for Caddy to reach it by DNS name.
  If Caddy returns 502, check that the service container is on the right network.

- **`cf-add-hostname.sh` needs env vars exported.** Use `set -a && source platform/.env && set +a`
  before running the script. The `.env` file has `CF_ACCOUNT_ID`, `CF_TUNNEL_ID`,
  `CF_API_EMAIL`, `CF_API_TOKEN`.

- **Don't map ports for Caddy-proxied services.** Docker DNS resolution on shared
  networks replaces the need for port mapping. Only use `ports:` for services that
  need direct access (e.g., Tailscale IP access like Memos on `100.107.36.2:5230`).

- **Caddy auto_https is off.** All `http://` in Caddyfile — Cloudflare handles HTTPS.

- **`forward_auth` breaks SPAs and API-heavy apps.** If a service has a JavaScript frontend
  that calls `/api/*` endpoints, Caddy's `forward_auth` will intercept ALL requests including
  API calls. The JS app gets empty/redirect responses instead of JSON → blank page.
  **Fix:** Use named matchers to exclude API paths from SSO:
  ```
  http://newservice.yourdomain.org {
      @api path /api/* /ws/*
      @web not path /api/* /ws/*

      handle @web {
          forward_auth authentik-server:9000 {
              uri /outpost.goauthentik.io/auth/caddy
              copy_headers X-authentik-username X-authentik-groups X-authentik-email
          }
          reverse_proxy newservice:8080
      }

      handle @api {
          reverse_proxy newservice:8080
      }
  }
  ```
  The service's own auth (e.g., Traccar login) handles API security.

- **Traccar OpenID is a Premium feature.** The self-hosted Traccar (open-source) exposes
  `openIdEnabled` in the API schema but the code doesn't exist. OIDC config keys in
  `traccar.xml` are silently ignored. Use Caddy `forward_auth` with API exclusions instead.

- **Authentik `forward_auth` requires a Proxy provider, not OAuth2.** When setting up SSO
  for Caddy's `forward_auth`, create a **Proxy provider** in Authentik (not OAuth2/OIDC).
  Then assign the Proxy provider to your Application AND add it to the Embedded Outpost's
  providers list. Without the outpost knowing about the provider, auth checks return 404.

- **Authentik API tokens may lack object-level permissions.** A token that can list
  applications may not be able to modify them. Check `system_permissions` via
  `/api/v3/core/users/me/`. If you hit "No X matches the given query" on PUT/PATCH,
  the token lacks the relevant permission — do it in the Authentik admin UI instead.

- **Authentik API endpoints use no hyphens.** The property mappings path is
  `propertymappings/provider/scope/` (NOT `property-mappings/`). Check available
  endpoints via the OpenAPI schema at `/api/v3/schema/`.

- **OIDC "Insufficient Scope" error.** Creating an OAuth2 provider in Authentik
  does NOT automatically assign OIDC scope mappings. The `property_mappings` list
  starts empty. You MUST assign at minimum: openid, email, profile.
  Find UUIDs: `GET /api/v3/propertymappings/provider/scope/`
  Assign: `PATCH /api/v3/providers/oauth2/<id>/` with `{"property_mappings": ["<uuid>",...]}`

- **CSRF 422 behind Cloudflare tunnel.** Apps (especially Rails) that receive plain
  HTTP from Caddy but the browser sends `Origin: https://` will reject the request as
  a CSRF mismatch. Fix: add `header_up X-Forwarded-Proto https` and
  `header_up X-Forwarded-Scheme https` in the Caddy reverse_proxy block. This tells
  the app the original request was HTTPS.

- **OIDC_REDIRECT_URI must be set explicitly.** Many apps (including Dawarich) default
  to `http://localhost:PORT/callback` for the OIDC redirect URI. This won't match
  what Authentik expects. Always set the full public URL:
  `OIDC_REDIRECT_URI: https://<subdomain>.yourdomain.org/<callback-path>`

- **Native OIDC apps don't need forward_auth.** If an app has built-in OIDC support
  (like Dawarich), use a plain reverse_proxy in Caddy — no forward_auth needed.
  forward_auth will conflict with the app's own auth flow and block API clients.

- **YAML healthcheck escaped quotes.** Complex healthcheck commands with deeply-nested
  escaped quotes break Docker Compose YAML parsing. Use simple `curl -sf` checks.

- **Docker Compose orphans.** After removing a service from compose, run
  `docker compose up -d --remove-orphans` to clean up stale containers.

- **Docker Compose can silently fail to start a service.** If `docker compose up -d <service>` produces no output and the container doesn't appear in `docker ps`, try `docker run -d --name <service> --restart unless-stopped --network platform-net <image>` directly. You can keep the compose definition for documentation/orphan-cleanup purposes — just note that the container was started manually. This happens occasionally with simple static-site services (nginx:alpine) where compose sees nothing to recreate.

- **Dev mount vs production build for static sites.** For sites where content changes frequently (e.g., a portfolio under active development), mount the source directory directly into the container instead of COPY+rebuild: `-v ~/src/project/public:/usr/share/nginx/html:ro`. This makes edits instant on the live site. Switch to COPY in the Dockerfile for production stability.
  OSM data before it starts serving. North America data takes 30-60 minutes.
  Container will show `health: starting` until import completes.

- **`cf-add-hostname.sh` env loading.** Running `source platform/.env` alone doesn't
  always export the vars. Use `set -a && source platform/.env && set +a` to ensure
  all variables are exported to the script's environment.

## Existing Services

| Service | Hostname | Compose | SSO | Notes |
|---------|----------|---------|-----|-------|
| Authentik | `auth.yourdomain.org`, `auth.example.com` | platform | N/A (it IS the auth provider) | Port 9000 |
| Forgejo | `git.yourdomain.org`, `git.example.org`, `git.example.com` | platform | No | SSH on port 2222 |
| Blog | `yourdomain.org`, `www.yourdomain.org` | platform | No | Currently 502 (blog container not on network) |
| Memos | Tailscale only (`100.107.36.2:5230`) | internal | No | No public hostname |
| Dawarich | `track.yourdomain.org` | internal | Native OIDC | GPS tracking, 4 containers (app+sidekiq+redis+postgis), replaced Traccar |
| Nominatim | `geo.yourdomain.org` | internal | No | Self-hosted reverse geocoding, North America data, port 8080 |
