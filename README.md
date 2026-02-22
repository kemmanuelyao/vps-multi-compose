# VPS Multi-Compose

Host multiple isolated Docker Compose stacks on a single VPS behind one entry point.

## Architecture

```
Internet :80/:443 → Traefik (gateway) → per-stack Nginx → stack services
                     TLS termination      path routing      (frontend, backend, etc.)
                     host routing          /api → backend
                                           /   → frontend
```

- **Traefik** handles TLS termination (Let's Encrypt) and hostname-based routing
- **Per-stack Nginx** handles path-based routing within each app
- New stacks self-register via Docker labels — no gateway config changes needed

## Project Structure

```
gateway/
  docker-compose.yml            # Traefik reverse proxy
  .env                          # ACME_EMAIL for Let's Encrypt
  letsencrypt/                  # ACME cert storage (gitignored)
apps/
  sample-app-nginx/                   # Nginx proxy approach (better isolation)
    docker-compose.yml          # proxy + frontend + backend
    proxy/nginx.conf            # Internal path routing
    frontend/                   # Static frontend files
  sample-app-traefik/           # Traefik-only approach (simpler, less isolation)
    docker-compose.yml          # frontend + backend (no proxy service)
    frontend/                   # Static frontend files
_template/                      # Starter template for new apps
```

## Two Routing Approaches

### 1. Nginx proxy per stack (`sample-app`)

```
Traefik → stack Nginx proxy → frontend / backend
```

- Only the Nginx proxy joins the `gateway` network; services stay on `internal`
- Full network isolation between stacks
- Path routing configured in `nginx.conf`

### 2. Traefik-only routing (`sample-app-traefik`)

```
Traefik → frontend / backend (directly)
```

- Every service joins the `gateway` network
- No per-stack proxy — path routing handled by Traefik labels
- Simpler setup, fewer containers, but less network isolation

## Networking

- **`gateway`** — External Docker network bridging Traefik to each stack's proxy. Created once, declared `external: true` in all compose files.
- **`internal`** — Per-stack bridge network (auto-created by Compose). Only the stack's Nginx proxy joins both networks. All other services stay on `internal` only for full isolation.

## Deployment

### Initial Setup

```bash
# Create the shared network (one-time)
docker network create gateway

# Start the gateway
cd gateway
docker compose up -d
```

### Start an App

```bash
cd apps/sample-app
docker compose up -d
```

Traefik auto-discovers the new stack — no restart needed.

### Verification

```bash
# HTTP → HTTPS redirect
curl -I http://sample.example.com

# Frontend
curl https://sample.example.com

# Backend routing
curl https://sample.example.com/api/

# Local testing (no DNS)
curl -H "Host: sample.example.com" http://localhost
```

## Adding a New Stack

```bash
# Copy the template
cp -r _template apps/my-app

# Create env file from example
cp apps/my-app/.env.example apps/my-app/.env

# Edit the files:
#   .env                - Set COMPOSE_PROJECT_NAME and APP_DOMAIN
#   docker-compose.yml  - Configure your services
#   proxy/nginx.conf    - Set up path routing

# Start it up
cd apps/my-app
docker compose up -d
```
