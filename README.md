# Distributed Monitoring Stack

Production-ready distributed monitoring solution for multiple Docker hosts using Prometheus, Loki, and Grafana.

## Architecture Overview

This monitoring stack uses a **distributed architecture** with:
- **Central monitoring server**: Single Grafana instance + Prometheus + Loki for data storage
- **Edge agents**: Lightweight collectors on each Docker host

### Components

```
┌─────────────────────────────────────────────────────────┐
│  Central Server (DigitalOcean Droplet)                  │
│  ┌──────────┐  ┌────────────┐  ┌──────┐  ┌───────┐    │
│  │ Grafana  │  │ Prometheus │  │ Loki │  │ Caddy │    │
│  └──────────┘  └────────────┘  └──────┘  └───────┘    │
│       │              │              │          │        │
└───────┼──────────────┼──────────────┼──────────┼────────┘
        │              │              │          │
        │         Remote Write    Remote Push   HTTPS/SSL
        │              │              │          │
┌───────┴──────────────┴──────────────┴──────────┴────────┐
│  Edge Hosts (Docker Servers)                             │
│  ┌───────────┐  ┌──────────┐  ┌──────────┐             │
│  │ Prom      │  │ Promtail │  │ Exporters│             │
│  │ Agent     │  │          │  │          │             │
│  └───────────┘  └──────────┘  └──────────┘             │
└──────────────────────────────────────────────────────────┘
```

## Repository Structure

This is an **umbrella repository** using Git submodules for each stack:

- **[monitoring-grafana](./grafana/)** - Central monitoring server stack
- **[monitoring-edge-basic](./edge-basic/)** - Edge agent for standard Docker hosts
- **[monitoring-edge-postgres](./edge-postgres/)** - Edge agent with PostgreSQL monitoring

Each submodule is a separate repository that can be deployed independently via Portainer GitOps.

## Quick Start

### 1. Central Server Deployment

Deploy on a dedicated DigitalOcean droplet:

```bash
# In Portainer, create new stack from Git:
Repository: https://github.com/YOUR-USERNAME/monitoring-grafana
Branch: main
Compose file: docker-compose.yml
```

Configure environment variables:
- `DOMAIN` - Your domain (e.g., monitoring.example.com)
- `BASIC_AUTH_USER` - Username for remote VPS authentication
- `BASIC_AUTH_PASSWORD` - Password (hashed with bcrypt)

### 2. Edge Agent Deployment

Deploy on each Docker host:

**For standard hosts:**
```bash
Repository: https://github.com/YOUR-USERNAME/monitoring-edge-basic
```

**For hosts with PostgreSQL:**
```bash
Repository: https://github.com/YOUR-USERNAME/monitoring-edge-postgres
```

Configure environment variables:
- `CENTRAL_PROMETHEUS_URL` - URL to central Prometheus (e.g., http://prometheus:9090)
- `CENTRAL_LOKI_URL` - URL to central Loki (e.g., http://loki:3100)
- `POSTGRES_DATA_SOURCE_NAME` - PostgreSQL connection string (postgres stack only)

## Features

### Metrics Collection
- **Retention**: 2 years on central server
- **Push model**: Prometheus remote write from edge to central
- **Real-time**: No polling delays
- **Zero edge storage**: Edge agents don't store metrics locally

### Log Aggregation
- **Retention**: 3 months on central server
- **Real-time streaming**: Promtail remote push to Loki
- **Disaster recovery**: Logs preserved even if containers are deleted
- **Centralized search**: All logs searchable from single Grafana instance

### Monitoring Targets
- Docker containers (cAdvisor)
- Host system metrics (Node Exporter)
- PostgreSQL/TimescaleDB (PostgreSQL Exporter with custom queries)
- Container logs (Promtail)

## Networking

### DigitalOcean Servers
Use **VPC (Virtual Private Cloud)** for free private networking between droplets in the same region.

See [docs/VPC-SETUP.md](./docs/VPC-SETUP.md) for setup instructions.

### Remote VPS (Outside DigitalOcean)
Use HTTPS with Basic Authentication via Caddy reverse proxy.

## Documentation

- [Architecture Details](./docs/ARCHITECTURE.md) - Deep dive into the design
- [Setup Guide](./docs/SETUP.md) - Step-by-step deployment with Portainer
- [Manual Deployment](./docs/MANUAL-DEPLOYMENT.md) - Deploy without Portainer using Docker Compose CLI
- [VPC Setup](./docs/VPC-SETUP.md) - DigitalOcean VPC configuration
- [Troubleshooting](./docs/TROUBLESHOOTING.md) - Common issues and solutions

## Submodules Setup

To clone this repository with all submodules:

```bash
git clone --recurse-submodules https://github.com/YOUR-USERNAME/monitoring-stack.git
```

To add submodules (for maintainers):

```bash
git submodule add https://github.com/YOUR-USERNAME/monitoring-grafana.git grafana
git submodule add https://github.com/YOUR-USERNAME/monitoring-edge-basic.git edge-basic
git submodule add https://github.com/YOUR-USERNAME/monitoring-edge-postgres.git edge-postgres
```

## Migration from Single-Host Setup

If you're migrating from the original single-host monitoring stack:

1. Deploy central stack first
2. Verify Grafana, Prometheus, and Loki are working
3. Deploy edge agents one host at a time
4. Import existing dashboards to new Grafana
5. Decommission old single-host stacks

## Requirements

- Docker 20.10+
- Docker Compose 2.0+
- Portainer (recommended for GitOps deployment)
- DigitalOcean VPC (optional, for free private networking)

## License

MIT License - see LICENSE file for details

## Original Project

This is a redesigned version of [server-docker-monitoring](https://github.com/oliveres/server-docker-monitoring), adapted for distributed multi-host deployments.
