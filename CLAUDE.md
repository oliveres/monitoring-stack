# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an **umbrella repository** for a distributed monitoring stack using Git submodules. The actual stack implementations are in separate repositories:

- `monitoring-grafana` - Central server with Grafana, Prometheus, Loki, Caddy
- `monitoring-edge-basic` - Edge agent with exporters and log collectors
- `monitoring-edge-postgres` - Edge agent with PostgreSQL monitoring

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
├── edge-basic/            # Git submodule → monitoring-edge-basic repo
└── edge-postgres/         # Git submodule → monitoring-edge-postgres repo
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
git submodule add https://github.com/USER/monitoring-grafana.git central
git submodule add https://github.com/USER/monitoring-edge-basic.git edge-basic
git submodule add https://github.com/USER/monitoring-edge-postgres.git edge-postgres
git commit -m "Add monitoring stack submodules"
```

### Update Submodules
```bash
# Update all submodules to latest
git submodule update --remote --merge

# Update specific submodule
cd central
git pull origin main
cd ..
git add central
git commit -m "Update central stack"
```

### Work on Submodule
```bash
# Enter submodule directory
cd central

# Make changes
vim docker-compose.yml

# Commit in submodule
git add .
git commit -m "Update Prometheus retention"
git push origin main

# Update umbrella repo to track new commit
cd ..
git add central
git commit -m "Track central stack update"
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
cd central
# Edit files (docker-compose.yml, configs, etc.)
git commit -am "Description of changes"
git push
# Portainer will auto-deploy if GitOps enabled
```

### Modify Edge Stack
```bash
cd edge-basic  # or edge-postgres
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

### Edge Stack (VPC)
```env
CENTRAL_PROMETHEUS_URL=http://10.116.0.2:9090/api/v1/write
CENTRAL_LOKI_URL=http://10.116.0.2:3100/loki/api/v1/push
HOSTNAME=edge-host-1
```

### Edge Stack (Remote via HTTPS)
```env
CENTRAL_PROMETHEUS_URL=https://monitoring.example.com/prometheus/api/v1/write
CENTRAL_LOKI_URL=https://monitoring.example.com/loki/api/v1/push
HOSTNAME=remote-vps-1
BASIC_AUTH_USER=remote
BASIC_AUTH_PASSWORD=password
```

### PostgreSQL Stack (Additional)
```env
POSTGRES_DATA_SOURCE_NAME=postgresql://user:pass@host:5432/db?sslmode=disable
```

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
