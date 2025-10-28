# Setup Guide

Complete step-by-step guide to deploy the distributed monitoring stack.

## Prerequisites

- Docker 20.10+ installed on all hosts
- Docker Compose 2.0+ installed
- Portainer running (for GitOps deployment)
- Domain name pointing to central server (for SSL)
- GitHub account (for hosting stack repositories)

## Overview

You'll deploy **3 separate Git repositories**:
1. `monitoring-grafana` - Central monitoring server
2. `monitoring-edge-basic` - Edge agents (standard hosts)
3. `monitoring-edge-postgres` - Edge agents (PostgreSQL hosts)

## Step 1: Create GitHub Repositories

Create three new repositories on GitHub:

```bash
# On GitHub, create these repositories:
1. monitoring-grafana (private recommended)
2. monitoring-edge-basic (private recommended)
3. monitoring-edge-postgres (private recommended)
```

Initialize each with the content from the respective submodule folders.

## Step 2: Deploy Central Stack

### 2.1 Prepare Central Server

Create a new DigitalOcean droplet:
- **Size**: Basic 4GB RAM / 2 vCPU ($24/month)
- **Region**: Choose your primary region (e.g., FRA1, NYC3)
- **Image**: Ubuntu 22.04 LTS
- **Additional**: Enable monitoring, IPv6, VPC

Install Docker and Portainer:
```bash
# Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER

# Install Portainer
docker volume create portainer_data
docker run -d -p 9443:9443 -p 8000:8000 \
  --name portainer --restart always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

### 2.2 Configure DNS

Point your domain to the central server:
```
monitoring.example.com  →  A record  →  <central-server-ip>
```

### 2.3 Deploy Central Stack in Portainer

1. Open Portainer: `https://<central-server-ip>:9443`
2. Go to **Stacks** → **Add stack**
3. Select **Git Repository**
4. Configure:
   - **Name**: `monitoring-grafana`
   - **Repository URL**: `https://github.com/YOUR-USERNAME/monitoring-grafana`
   - **Branch**: `main`
   - **Compose path**: `docker-compose.yml`
   - **Authentication**: Configure if private repo

5. **Environment variables**:
   ```env
   DOMAIN=monitoring.example.com
   GRAFANA_ADMIN_USER=admin
   GRAFANA_ADMIN_PASSWORD=your-secure-password
   BASIC_AUTH_USER=remote
   BASIC_AUTH_PASSWORD_HASH=$2a$12$... (bcrypt hash)
   ```

6. Enable **GitOps** (optional):
   - Polling interval: 5 minutes
   - Auto-update on changes

7. Click **Deploy the stack**

### 2.4 Generate Basic Auth Password Hash

For remote VPS authentication:

```bash
# Install htpasswd
sudo apt-get install apache2-utils

# Generate bcrypt hash
htpasswd -nbB remote your-password

# Copy the hash (everything after "remote:")
# Example output: remote:$2y$05$...
# Use only: $2y$05$...
```

### 2.5 Verify Central Stack

```bash
# Check all containers are running
docker ps

# Expected containers:
# - monitoring-caddy
# - monitoring-grafana
# - monitoring-prometheus
# - monitoring-loki

# Check Caddy SSL
curl https://monitoring.example.com
# Should redirect to Grafana login

# Check logs
docker logs monitoring-caddy
docker logs monitoring-prometheus
docker logs monitoring-loki
```

### 2.6 Access Grafana

1. Open: `https://monitoring.example.com`
2. Login with admin credentials
3. Verify datasources:
   - Prometheus (should be green)
   - Loki (should be green)

## Step 3: Setup DigitalOcean VPC (Optional but Recommended)

See [VPC-SETUP.md](./VPC-SETUP.md) for detailed instructions.

**Quick summary:**
1. Go to DigitalOcean → Networking → VPC
2. Create new VPC in your region
3. Assign central server to VPC
4. Assign edge servers to same VPC
5. Update docker-compose environment variables with VPC private IPs

## Step 4: Deploy Edge Stack (DigitalOcean Hosts)

For each DigitalOcean droplet with Docker:

### 4.1 Deploy via Portainer

1. Open Portainer on the edge host
2. Go to **Stacks** → **Add stack**
3. Select **Git Repository**
4. Configure:
   - **Name**: `monitoring-edge`
   - **Repository URL**:
     - Standard: `https://github.com/YOUR-USERNAME/monitoring-edge-basic`
     - With PostgreSQL: `https://github.com/YOUR-USERNAME/monitoring-edge-postgres`
   - **Branch**: `main`

5. **Environment variables for VPC setup**:
   ```env
   CENTRAL_PROMETHEUS_URL=http://10.XXX.0.2:9090/api/v1/write
   CENTRAL_LOKI_URL=http://10.XXX.0.2:3100/loki/api/v1/push
   HOSTNAME=edge-host-1
   ```

6. **For PostgreSQL stack, add**:
   ```env
   POSTGRES_DATA_SOURCE_NAME=postgresql://user:password@host:5432/dbname?sslmode=disable
   ```

7. Enable **GitOps** and deploy

### 4.2 Verify Edge Stack

```bash
# Check containers
docker ps

# Expected containers (basic):
# - monitoring-prometheus
# - monitoring-promtail
# - monitoring-cadvisor
# - monitoring-node-exporter

# Plus for postgres:
# - monitoring-postgres-exporter

# Check Prometheus is sending data
docker logs monitoring-prometheus
# Should see: "Remote write succeeded"

# Check Promtail is sending logs
docker logs monitoring-promtail
# Should see: "Successfully sent batch"
```

## Step 5: Deploy Edge Stack (Remote VPS)

For VPS outside DigitalOcean (HTTPS + Basic Auth):

### 5.1 Deploy via Portainer

Same as Step 4, but different environment variables:

```env
CENTRAL_PROMETHEUS_URL=https://monitoring.example.com/prometheus/api/v1/write
CENTRAL_LOKI_URL=https://monitoring.example.com/loki/api/v1/push
HOSTNAME=remote-vps-1
BASIC_AUTH_USER=remote
BASIC_AUTH_PASSWORD=your-password
```

The Caddy reverse proxy on central server handles authentication.

## Step 6: Verify Data Collection

### 6.1 Check Metrics in Grafana

1. Open Grafana: `https://monitoring.example.com`
2. Go to **Explore**
3. Select **Prometheus** datasource
4. Run query:
   ```promql
   up{job="cadvisor"}
   ```
5. Should see metrics from all edge hosts

### 6.2 Check Logs in Grafana

1. Go to **Explore**
2. Select **Loki** datasource
3. Run query:
   ```logql
   {job="docker"}
   ```
4. Should see logs from all containers across all hosts

### 6.3 Import Dashboards

Pre-configured dashboards will be auto-provisioned. To import additional:

1. Go to **Dashboards** → **Import**
2. Import ID `15120` (Docker monitoring)
3. Import ID `9628` (PostgreSQL monitoring)
4. Configure datasources when prompted

## Step 7: Configure Alerting (Optional)

### 7.1 Add Alertmanager to Central Stack

Edit `monitoring-grafana/docker-compose.yml`:

```yaml
  alertmanager:
    image: prom/alertmanager:latest
    container_name: monitoring-alertmanager
    restart: unless-stopped
    networks:
      - monitoring
    expose:
      - 9093
    volumes:
      - ./alertmanager:/etc/alertmanager
      - alertmanager-data:/alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
```

### 7.2 Configure Alert Rules

Create `prometheus/alerts.yml`:

```yaml
groups:
  - name: host_alerts
    interval: 30s
    rules:
      - alert: HostDown
        expr: up{job="node-exporter"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Host {{ $labels.instance }} is down"

      - alert: HighMemoryUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 90
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Memory usage above 90% on {{ $labels.instance }}"
```

## Troubleshooting

### Central Stack Issues

**Caddy SSL fails:**
```bash
# Check DNS
dig monitoring.example.com

# Check Caddy logs
docker logs monitoring-caddy

# Common issue: Port 80/443 not open
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

**Prometheus remote write receiver not working:**
```bash
# Check if endpoint is accessible
curl -X POST http://localhost:9090/api/v1/write

# Check Prometheus config
docker exec monitoring-prometheus cat /etc/prometheus/prometheus.yml
```

**Loki not receiving logs:**
```bash
# Check Loki is running
docker logs monitoring-loki

# Test push endpoint
curl -X POST http://localhost:3100/loki/api/v1/push
```

### Edge Stack Issues

**Prometheus remote write failing:**
```bash
# Check logs
docker logs monitoring-prometheus

# Common errors:
# - "connection refused" → Check CENTRAL_PROMETHEUS_URL
# - "401 unauthorized" → Check Basic Auth credentials
# - "certificate error" → Check domain SSL

# Test connection manually
curl -v https://monitoring.example.com/prometheus/api/v1/write
```

**Promtail not sending logs:**
```bash
# Check Promtail logs
docker logs monitoring-promtail

# Check Docker socket permissions
ls -l /var/run/docker.sock

# Verify Promtail can read logs
docker exec monitoring-promtail ls /var/lib/docker/containers
```

**cAdvisor not collecting metrics:**
```bash
# Check cAdvisor is running in privileged mode
docker inspect monitoring-cadvisor | grep Privileged

# Check cgroup is enabled (see TROUBLESHOOTING.md)
cat /proc/cmdline | grep cgroup
```

### Network Issues

**VPC hosts can't reach central:**
```bash
# Verify VPC membership
# In DigitalOcean console: Networking → VPC → Check members

# Test connectivity
ping 10.XXX.0.2  # Central server VPC IP

# Check if Prometheus is listening on VPC IP
docker exec monitoring-prometheus netstat -tlnp | grep 9090
```

**Remote VPS authentication failing:**
```bash
# Test Basic Auth
curl -u remote:password https://monitoring.example.com/prometheus/api/v1/write

# Verify Caddy is configured for Basic Auth
docker exec monitoring-caddy cat /etc/caddy/Caddyfile
```

## Maintenance

### Update Stacks

With GitOps enabled, Portainer auto-updates when you push to Git.

Manual update:
1. Go to Portainer → Stacks → Your stack
2. Click **Pull and redeploy**

### Backup Grafana Dashboards

```bash
# Export dashboard JSON
# Grafana UI → Dashboard → Settings → JSON Model

# Or use API
curl -H "Authorization: Bearer <api-key>" \
  https://monitoring.example.com/api/dashboards/uid/<dashboard-uid>
```

### Monitor Disk Usage

```bash
# Check Prometheus data size
du -sh /var/lib/docker/volumes/monitoring-grafana_prometheus-data

# Check Loki data size
du -sh /var/lib/docker/volumes/monitoring-grafana_loki-data

# Set up alerts for disk usage in Grafana
```

## Next Steps

1. Configure alerting rules for your use case
2. Create custom dashboards
3. Set up automated backups (Prometheus snapshots)
4. Consider adding Grafana authentication (OAuth, LDAP)
5. Implement log retention policies in Loki

## Getting Help

- Check [ARCHITECTURE.md](./ARCHITECTURE.md) for design details
- See [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) for common issues
- Review logs: `docker logs <container-name>`
- GitHub Issues: Report problems in respective repo
