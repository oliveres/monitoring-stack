# Architecture Documentation

## Overview

This monitoring stack implements a **hub-and-spoke architecture** with a central data collection point and lightweight edge agents on each Docker host.

## Design Principles

1. **Zero data loss**: All metrics and logs pushed immediately to central server
2. **Minimal edge footprint**: Edge agents don't store data locally
3. **Single pane of glass**: One Grafana instance for all hosts
4. **GitOps ready**: Each stack deployable via Portainer from Git
5. **Cost optimized**: Use DigitalOcean VPC for free inter-droplet traffic

## Component Architecture

### Central Stack

**Location**: Dedicated DigitalOcean droplet (recommended: 4GB RAM, 2 vCPU minimum)

```
┌─────────────────────────────────────────┐
│  Central Monitoring Server               │
│                                          │
│  ┌────────────────────────────────┐    │
│  │ Caddy (Port 80/443)            │    │
│  │  - Automatic SSL (Let's Encrypt) │   │
│  │  - Reverse proxy                │   │
│  │  - Basic Auth for remote VPS   │   │
│  └──────┬──────────────────────────┘   │
│         │                               │
│  ┌──────▼──────┐  ┌────────┐  ┌────┐  │
│  │  Grafana    │  │ Prom   │  │Loki│  │
│  │  :3000      │  │ :9090  │  │:3100│ │
│  └─────────────┘  └───┬────┘  └──┬─┘  │
│                       │          │     │
│                  Remote Write  Remote  │
│                   Receiver     Push    │
│                       │          │     │
└───────────────────────┼──────────┼─────┘
                        │          │
                   VPC or HTTPS    │
                        │          │
```

**Services:**
- **Caddy**:
  - Automatic SSL certificate management
  - Reverse proxy for Grafana
  - Basic Authentication for remote VPS connections
  - Port: 80 (redirect), 443 (HTTPS)

- **Prometheus**:
  - Time-series database for metrics
  - Remote write receiver for edge agents
  - Retention: 2 years
  - Downsampling: Optional (can be added later)
  - Port: 9090 (internal or VPC)

- **Loki**:
  - Log aggregation system
  - Remote push receiver for Promtail
  - Retention: 3 months
  - Port: 3100 (internal or VPC)

- **Grafana**:
  - Visualization and dashboards
  - Auto-provisioned datasources (Prometheus + Loki)
  - Auto-provisioned dashboards
  - Port: 3000 (behind Caddy proxy)

**Data Volumes:**
- `prometheus-data`: Metrics storage (~1-5GB per host per year)
- `loki-data`: Log storage (~500MB-2GB per host per month)
- `grafana-data`: Dashboards and config
- `caddy-data`: SSL certificates

### Edge Stack (Basic)

**Location**: Each Docker host without PostgreSQL

```
┌─────────────────────────────────────────┐
│  Docker Host (Edge)                      │
│                                          │
│  ┌────────────┐  ┌────────────┐        │
│  │ Prometheus │  │ Promtail   │        │
│  │ Agent      │  │            │        │
│  │ (0 storage)│  │            │        │
│  └──────┬─────┘  └──────┬─────┘        │
│         │                │              │
│    ┌────▼────┐      ┌───▼────┐         │
│    │ cAdvisor│      │ Docker │         │
│    └─────────┘      │ Socket │         │
│    ┌────────────┐   └────────┘         │
│    │ Node       │                       │
│    │ Exporter   │                       │
│    └────────────┘                       │
│         │                │              │
│    Scrapes Metrics   Reads Logs        │
│         │                │              │
└─────────┼────────────────┼──────────────┘
          │                │
     Remote Write      Remote Push
          │                │
          ▼                ▼
    Central Prometheus  Central Loki
```

**Services:**
- **Prometheus Agent**:
  - Scrapes local exporters
  - Immediately forwards via remote write
  - Zero local storage (--storage.tsdb.retention.time=0)
  - Minimal memory footprint

- **Promtail**:
  - Tails Docker container logs
  - Pushes to central Loki in real-time
  - Maintains position tracking

- **cAdvisor**:
  - Container metrics (CPU, memory, network, disk)
  - Scraped every 5-10 seconds

- **Node Exporter**:
  - Host system metrics (CPU, memory, disk, network)
  - Scraped every 10-15 seconds

### Edge Stack (PostgreSQL)

**Location**: Docker hosts with PostgreSQL/TimescaleDB

Same as Edge Basic, plus:

```
┌──────────────────────┐
│ PostgreSQL Exporter  │
│                      │
│  Connected to:       │
│  - dispatch-network  │
│  - internal network  │
└──────────────────────┘
         │
    Scrapes PG metrics
         │
    (via Prometheus)
```

**Additional Service:**
- **PostgreSQL Exporter**:
  - Custom TimescaleDB queries
  - Monitors hypertable chunks, compression, jobs
  - Connected to both internal and external networks
  - Environment: `DATA_SOURCE_NAME` connection string

## Data Flow

### Metrics Flow

```
Docker Container
    │
    ▼
cAdvisor (collects container metrics)
    │
    ▼
Prometheus Agent (scrapes)
    │
    ▼ Remote Write (HTTP)
Central Prometheus (stores)
    │
    ▼ Query (PromQL)
Grafana (visualizes)
```

**Characteristics:**
- Push frequency: Real-time (15-30s batches)
- Data format: Prometheus remote write protocol
- Compression: Snappy compression
- Authentication: Basic Auth or VPC private network

### Logs Flow

```
Docker Container
    │
    ▼
Docker Socket (/var/log/docker/containers)
    │
    ▼
Promtail (reads log files)
    │
    ▼ Remote Push (HTTP)
Central Loki (stores)
    │
    ▼ Query (LogQL)
Grafana (visualizes)
```

**Characteristics:**
- Push frequency: Real-time (< 1s delay)
- Data format: JSON over HTTP
- Compression: Gzip
- Labels: container name, host, job

## Networking

### Option 1: DigitalOcean VPC (Recommended for DO hosts)

```
┌─────────────────────────────────────────┐
│  DigitalOcean VPC (10.XXX.0.0/16)       │
│                                          │
│  ┌──────────┐  ┌──────────┐  ┌────────┐│
│  │ Central  │  │ Edge 1   │  │ Edge 2 ││
│  │          │  │          │  │        ││
│  └──────────┘  └──────────┘  └────────┘│
│                                          │
│  Private IPs: 10.XXX.0.X                │
│  No egress charges                       │
└─────────────────────────────────────────┘
          │
          ▼ Public IP
     Internet (Grafana UI)
```

**Benefits:**
- Zero cost for inter-droplet traffic
- Low latency
- Secure private network
- No authentication needed between DO hosts

### Option 2: HTTPS + Basic Auth (For remote VPS)

```
┌────────────┐                  ┌──────────┐
│ Remote VPS │                  │ Central  │
│            │                  │ DO Host  │
│  Edge      │  ──────────────> │          │
│  Stack     │   HTTPS + Auth   │  Caddy   │
│            │                  │  Proxy   │
└────────────┘                  └──────────┘

Internet (encrypted)
```

**Security:**
- TLS 1.3 encryption
- Basic Authentication (bcrypt hashed passwords)
- Rate limiting (optional, via Caddy)
- IP whitelisting (optional, via Caddy)

## Scalability

### Current Design (5-10 hosts)
- Single Prometheus instance
- Single Loki instance
- Suitable for: 50-100 containers total

### Future Scaling (10-50 hosts)
**Option 1: Vertical scaling**
- Upgrade central server (8GB+ RAM)
- Enable Prometheus downsampling
- Loki with object storage backend (S3)

**Option 2: Horizontal scaling**
- Prometheus federation (hierarchical)
- Loki distributed mode (read/write separation)
- Multiple Grafana instances (optional)

### Performance Estimates

**Per Docker host (10 containers):**
- Metrics: ~100MB/day uncompressed → ~20MB/day stored
- Logs: ~50-200MB/day (depends on verbosity)

**Central server storage (5 hosts, 2 years metrics, 3 months logs):**
- Prometheus: ~5 hosts × 365 days × 2 years × 20MB = ~73GB
- Loki: ~5 hosts × 90 days × 100MB = ~45GB
- Total: ~120GB storage

**Recommended central server specs:**
- 4GB RAM (minimum), 8GB (recommended)
- 2 vCPU (minimum), 4 vCPU (recommended)
- 150GB SSD storage
- Cost: ~$24-48/month on DigitalOcean

## High Availability Considerations

Current design: **Single point of failure** (acceptable for 5-10 hosts)

For production HA:
1. **Prometheus**: Pair with identical instances + Thanos for deduplication
2. **Loki**: Run in microservices mode with object storage
3. **Grafana**: Load balanced instances (shared database)

## Security Best Practices

1. **VPC Usage**:
   - Use for all DigitalOcean droplets in same region
   - Bind Prometheus/Loki to VPC private IPs only

2. **Authentication**:
   - Strong passwords for Grafana (change default admin/admin)
   - Bcrypt hashed passwords for Basic Auth
   - Consider OAuth for Grafana (Google, GitHub)

3. **Network Exposure**:
   - Only Caddy (443) exposed to internet
   - Prometheus/Loki not publicly accessible
   - Use firewall rules (DigitalOcean Cloud Firewall)

4. **Secrets Management**:
   - Use Portainer environment variables
   - Never commit .env files to Git
   - Rotate passwords regularly

## Monitoring the Monitors

Self-monitoring is built-in:
- Prometheus scrapes itself
- Grafana monitors Prometheus/Loki health
- Alerting can be added via Prometheus Alertmanager

## Disaster Recovery

**Data backup strategies:**
1. **Metrics**: Prometheus snapshots (optional)
2. **Logs**: Loki with S3 backend (future enhancement)
3. **Dashboards**: Auto-provisioned via Git (GitOps)
4. **Configuration**: All in Git repositories

**Recovery Time Objective (RTO):**
- Redeploy from Git: ~5-10 minutes
- Data loss: None (if central server survives)

## Cost Analysis

**DigitalOcean setup (5 hosts):**
- Central server (4GB): $24/month
- Edge agents: $0 (use existing hosts)
- VPC traffic: $0
- Remote VPS traffic: ~$1-2/month (egress)
- **Total: ~$25-26/month**

**Comparable SaaS alternatives:**
- Datadog: ~$150-300/month (5 hosts)
- Grafana Cloud: ~$100-200/month
- **Savings: ~75-85% cost reduction**
