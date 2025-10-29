# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an **umbrella repository** for a distributed monitoring stack using Git submodules. The actual stack implementations are in separate repositories:

- `monitoring-grafana` - Central server with Grafana, Prometheus, Loki, Caddy
- `monitoring-edge` - Unified edge agent with optional exporters (PostgreSQL, PgBouncer, Caddy)

Each submodule is deployed independently via Portainer GitOps to different servers.

## Repository Structure

```
monitoring-stack/           # This umbrella repo (documentation only)
├── docs/                   # Architecture and setup guides
│   ├── ARCHITECTURE.md     # Design details
│   ├── SETUP.md           # Deployment instructions
│   ├── VPC-SETUP.md       # DigitalOcean VPC configuration
│   └── TROUBLESHOOTING.md # Common issues
├── grafana/               # Git submodule → monitoring-grafana repo
└── edge/                  # Git submodule → monitoring-edge repo (unified)
```

## Important Notes

1. **This is documentation-only repository**: No Docker stacks run from here
2. **Submodules are separate repos**: Each has own git history and deployment
3. **Changes to stacks**: Must be made in respective submodule repositories
4. **Portainer deployment**: Each stack deployed from its own GitHub repo

## Working with Submodules

### Clone with Submodules
```bash
git clone --recurse-submodules https://github.com/USER/monitoring-stack.git
```

### Add Submodules (First Time Setup)
```bash
git submodule add https://github.com/USER/monitoring-grafana.git grafana
git submodule add https://github.com/USER/monitoring-edge.git edge
git commit -m "Add monitoring stack submodules"
```

### Update Submodules
```bash
# Update all submodules to latest
git submodule update --remote --merge

# Update specific submodule
cd grafana  # or edge
git pull origin main
cd ..
git add grafana
git commit -m "Update grafana stack"
```

### Work on Submodule
```bash
# Enter submodule directory
cd grafana  # or edge

# Make changes
vim docker-compose.yml

# Commit in submodule
git add .
git commit -m "Update Prometheus retention"
git push origin main

# Update umbrella repo to track new commit
cd ..
git add grafana
git commit -m "Track grafana stack update"
git push
```

## Architecture Overview

**Deployment Model**: Hub-and-spoke
- **Central server**: Single Grafana + Prometheus + Loki (2-4GB RAM)
- **Edge agents**: Lightweight collectors on each Docker host (minimal resources)

**Data Flow**:
- Metrics: Edge Prometheus → Remote Write → Central Prometheus
- Logs: Edge Promtail → Remote Push → Central Loki
- Visualization: Central Grafana queries both datasources

**Networking**:
- DigitalOcean hosts: VPC private network (free transfer)
- Remote VPS: HTTPS + Basic Auth via Caddy

## Key Technologies

- **Prometheus**: Metrics storage (2 years retention on central)
- **Loki**: Log aggregation (3 months retention)
- **Grafana**: Visualization dashboard
- **Caddy**: Reverse proxy with automatic SSL
- **Promtail**: Log collector
- **cAdvisor**: Container metrics
- **Node Exporter**: Host metrics
- **PostgreSQL Exporter**: Database metrics (where applicable)

## Common Tasks

### Modify Central Stack
```bash
cd grafana
# Edit files (docker-compose.yml, configs, etc.)
git commit -am "Description of changes"
git push
# Portainer will auto-deploy if GitOps enabled
```

### Modify Edge Stack
```bash
cd edge
# Edit files
git commit -am "Description of changes"
git push
# All edge hosts with GitOps will auto-update
```

### Update Documentation
```bash
# Directly in umbrella repo
cd docs
vim SETUP.md
git commit -am "Update setup instructions"
git push
```

### Add New Dashboard
```bash
cd grafana/grafana/provisioning/dashboards
# Add new .json dashboard file
git add new-dashboard.json
git commit -m "Add new dashboard"
git push
# Redeploy central stack in Portainer
```

## Environment Variables

Each stack uses environment variables configured in Portainer:

### Central Stack
```env
DOMAIN=monitoring.example.com
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=secure-password
BASIC_AUTH_USER=remote
BASIC_AUTH_PASSWORD_HASH=$2a$12$...
VPC_PRIVATE_IP=10.116.0.2
```

### Edge Stack - Core (Required)
```env
# Host identification
HOSTNAME=edge-host-1

# Central endpoints
CENTRAL_PROMETHEUS_URL=http://10.116.0.2:9090/api/v1/write  # VPC
# OR for remote HTTPS:
# CENTRAL_PROMETHEUS_URL=https://monitoring.example.com/prometheus/api/v1/write
CENTRAL_LOKI_URL=http://10.116.0.2:3100/loki/api/v1/push    # VPC
# OR for remote HTTPS:
# CENTRAL_LOKI_URL=https://monitoring.example.com/loki/api/v1/push

# Basic auth (only for HTTPS endpoints)
BASIC_AUTH_USER=remote
BASIC_AUTH_PASSWORD=password
```

### Edge Stack - Optional Exporters

Enable optional components using Docker Compose profiles:

```env
# Activate profiles (comma-separated)
COMPOSE_PROFILES=postgres,pgbouncer,caddy

# PostgreSQL/TimescaleDB (requires 'postgres' profile)
POSTGRES_DATA_SOURCE_NAME=postgresql://user:pass@host:5432/db?sslmode=disable
POSTGRES_EXPORTER_TARGET=postgres-exporter:9187
PROD_NETWORK=production-network  # External Docker network (reaches PostgreSQL, PgBouncer, Caddy, etc.)

# PgBouncer (requires 'pgbouncer' profile)
PGBOUNCER_EXPORTER_CONN_STRING=postgresql://user:pass@host:6432/pgbouncer?sslmode=disable
PGBOUNCER_EXPORTER_TARGET=pgbouncer-exporter:9127

# Caddy metrics (requires 'caddy' profile - scrapes external Caddy)
CADDY_METRICS_TARGET=localhost:2019
```

## Docker Compose Profiles (Edge Stack)

The unified edge stack uses Docker Compose profiles to enable optional exporters:

### Available Profiles

| Profile     | Components | Use Case |
|-------------|------------|----------|
| (baseline)  | Prometheus, Promtail, cAdvisor, Node Exporter | Basic host and container monitoring |
| `postgres`  | + PostgreSQL Exporter | PostgreSQL/TimescaleDB databases |
| `pgbouncer` | + PgBouncer Exporter | PgBouncer connection pooler |
| `caddy`     | + Caddy scrape job | External Caddy web server |

### Activation

**In Portainer:**
1. Add stack from Git repository
2. Set environment variable: `COMPOSE_PROFILES=postgres,pgbouncer` (comma-separated)
3. Configure required env vars for each profile (see Environment Variables section)
4. Deploy

**Via CLI:**
```bash
# Baseline only
docker compose up -d

# With PostgreSQL
COMPOSE_PROFILES=postgres docker compose up -d

# Multiple profiles
COMPOSE_PROFILES=postgres,pgbouncer,caddy docker compose up -d
```

### Important Notes

- **Baseline deployment** (no profiles) includes: Prometheus agent, Promtail, cAdvisor, Node Exporter
- **Optional components** start only when their profile is active AND target env var is set
- **External networks** (e.g., `PROD_NETWORK`) must exist before deploying with profiles
- **Caddy profile** does NOT launch Caddy container - it only enables scraping of existing Caddy instance

## Deployment Workflow

1. **Create GitHub repositories** for each stack
2. **Push stack code** to respective repositories
3. **Configure Portainer** on each server:
   - Add stack from Git
   - Set environment variables
   - Enable GitOps (optional)
4. **Deploy stacks**:
   - Central first (verify Grafana works)
   - Edge stacks second (verify data flowing)
5. **Verify monitoring** in Grafana

## Troubleshooting

- Check [docs/TROUBLESHOOTING.md](./docs/TROUBLESHOOTING.md) first
- View container logs: `docker logs monitoring-<service>`
- Test connectivity: `curl http://grafana-ip:9090/-/healthy`
- Verify environment variables in Portainer

## Migration from Single-Host Setup

If migrating from original `server-docker-monitoring`:

1. Deploy central stack on new dedicated server
2. Verify Grafana/Prometheus/Loki operational
3. Deploy edge stacks on each Docker host
4. Import existing dashboards to new Grafana
5. Verify data collection from all hosts
6. Decommission old single-host stacks

## Important Reminders

- **Never commit .env files** with secrets
- **Use Portainer environment variables** for secrets
- **Test in staging** before production changes
- **GitOps enabled** means auto-deploy on push - be careful!
- **VPC setup** saves significant bandwidth costs on DigitalOcean
- **SSL certificates** managed automatically by Caddy (Let's Encrypt)

## Development Best Practices

1. **Make changes in submodule repos**, not umbrella repo
2. **Test locally** with docker-compose before pushing
3. **Use GitOps carefully** - pushes trigger auto-deploys
4. **Document configuration changes** in respective README files
5. **Keep retention policies realistic** to control disk usage
6. **Monitor the monitors** - watch Prometheus/Loki disk usage

## Getting Help

- Review [docs/ARCHITECTURE.md](./docs/ARCHITECTURE.md) for design details
- Check [docs/SETUP.md](./docs/SETUP.md) for deployment steps
- See [docs/VPC-SETUP.md](./docs/VPC-SETUP.md) for DigitalOcean VPC
- Consult [docs/TROUBLESHOOTING.md](./docs/TROUBLESHOOTING.md) for issues
- Open GitHub issue in respective stack repository

## Original Project

Based on [server-docker-monitoring](https://github.com/oliveres/server-docker-monitoring), redesigned for multi-host distributed deployment.
