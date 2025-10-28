# Getting Started Guide

Quick start guide to deploy the distributed monitoring stack in 30 minutes.

## What You'll Build

```
                     ┌─────────────────┐
                     │   Central       │
                     │   Monitoring    │
                     │   Server        │
                     │   (Grafana UI)  │
                     └────────┬────────┘
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
    ┌─────▼─────┐       ┌─────▼─────┐      ┌─────▼─────┐
    │  Edge     │       │  Edge     │      │  Edge     │
    │  Host 1   │       │  Host 2   │      │  Host 3   │
    │  (Docker) │       │  (Docker) │      │  (+PG)    │
    └───────────┘       └───────────┘      └───────────┘
```

- **1 Central Server**: Grafana + Prometheus + Loki
- **Multiple Edge Hosts**: Lightweight metrics & log collectors

## Prerequisites Checklist

- [ ] Docker 20.10+ on all servers
- [ ] Docker Compose 2.0+ on all servers
- [ ] Portainer installed on all servers
- [ ] Domain name for central server (e.g., `monitoring.example.com`)
- [ ] GitHub account for hosting repositories
- [ ] DigitalOcean account (if using VPC) OR access to any cloud/VPS provider

## Quick Start (30 Minutes)

### Phase 1: Prepare Repositories (10 min)

**1. Create GitHub Repositories**

Create 4 new repositories on GitHub:
- `monitoring-stack` (umbrella)
- `monitoring-central`
- `monitoring-edge-basic`
- `monitoring-edge-postgres`

**2. Push Code**

```bash
# From your local monitoring-stack directory
cd central
git init && git add . && git commit -m "Initial commit"
git remote add origin https://github.com/YOUR-USERNAME/monitoring-central.git
git push -u origin main

cd ../edge-basic
git init && git add . && git commit -m "Initial commit"
git remote add origin https://github.com/YOUR-USERNAME/monitoring-edge-basic.git
git push -u origin main

cd ../edge-postgres
git init && git add . && git commit -m "Initial commit"
git remote add origin https://github.com/YOUR-USERNAME/monitoring-edge-postgres.git
git push -u origin main
```

See [docs/GIT-SETUP.md](./docs/GIT-SETUP.md) for detailed instructions.

### Phase 2: Deploy Central Server (10 min)

**1. Provision Server**

Create a DigitalOcean droplet:
- **Size**: 4GB RAM / 2 vCPU ($24/month)
- **Region**: Your preferred region (e.g., FRA1)
- **Image**: Ubuntu 22.04 LTS
- Enable VPC

**2. Point Domain to Server**

```bash
# Add A record to your DNS:
monitoring.example.com → <server-public-ip>
```

Wait 5-10 minutes for DNS propagation.

**3. Deploy via Portainer**

1. Open Portainer: `https://<server-ip>:9443`
2. Go to **Stacks** → **Add stack**
3. Name: `monitoring-central`
4. Git Repository: `https://github.com/YOUR-USERNAME/monitoring-central`
5. Add environment variables:
   ```
   DOMAIN=monitoring.example.com
   GRAFANA_ADMIN_USER=admin
   GRAFANA_ADMIN_PASSWORD=your-secure-password
   PROMETHEUS_RETENTION=2y
   ```
6. For remote VPS access, also add:
   ```bash
   # Generate hash:
   htpasswd -nbB remote yourpassword
   # Copy hash (after "remote:") and add:
   BASIC_AUTH_USER=remote
   BASIC_AUTH_PASSWORD_HASH=$2a$12$...your-hash...
   ```
7. Click **Deploy the stack**

**4. Verify**

```bash
# Check containers
docker ps

# Open Grafana
https://monitoring.example.com
# Login with admin credentials
```

✅ **Checkpoint**: Grafana should be accessible with green Prometheus and Loki datasources.

### Phase 3: Deploy Edge Agents (10 min)

**For DigitalOcean Hosts (Using VPC)**

1. **Setup VPC**:
   - DigitalOcean console → Networking → VPC
   - Create VPC in your region
   - Assign central + edge servers to VPC
   - Note VPC private IPs (e.g., `10.116.0.2` for central)

2. **Deploy Edge Stack** (on each Docker host):
   - Portainer → Stacks → Add stack
   - Git: `https://github.com/YOUR-USERNAME/monitoring-edge-basic`
   - Environment:
     ```
     HOSTNAME=edge-host-1
     CENTRAL_PROMETHEUS_URL=http://10.116.0.2:9090/api/v1/write
     CENTRAL_LOKI_URL=http://10.116.0.2:3100/loki/api/v1/push
     ```
   - Deploy

**For Remote VPS (HTTPS)**

- Same steps, but use:
  ```
  HOSTNAME=remote-vps-1
  CENTRAL_PROMETHEUS_URL=https://monitoring.example.com/prometheus/api/v1/write
  CENTRAL_LOKI_URL=https://monitoring.example.com/loki/api/v1/push
  BASIC_AUTH_USER=remote
  BASIC_AUTH_PASSWORD=yourpassword
  ```

**For PostgreSQL Hosts**

- Use `monitoring-edge-postgres` repo
- Also add:
  ```
  POSTGRES_DATA_SOURCE_NAME=postgresql://user:pass@host:5432/db?sslmode=disable
  ```

**3. Verify Edge Hosts**

```bash
# On edge host, check logs
docker logs monitoring-prometheus | grep "Remote write succeeded"
docker logs monitoring-promtail | grep "Successfully sent"
```

✅ **Checkpoint**: Edge hosts sending data to central server.

### Phase 4: Verify Monitoring (5 min)

**1. Check Metrics in Grafana**

```
https://monitoring.example.com
→ Explore
→ Prometheus
→ Query: up{job="cadvisor"}
```

Should see metrics from all edge hosts.

**2. Check Logs in Grafana**

```
→ Explore
→ Loki
→ Query: {job="docker"}
```

Should see logs from all containers across all hosts.

**3. Import Dashboards**

```
→ Dashboards → Import
→ ID: 15120 (Docker monitoring)
→ Select Prometheus datasource
→ Import
```

✅ **Success!** You now have distributed monitoring across all hosts.

## What's Next?

### Immediate Actions

1. **Change default password**: Update Grafana admin password
2. **Set up alerts**: Configure Prometheus alerting rules
3. **Create dashboards**: Build custom dashboards for your apps
4. **Enable GitOps**: Turn on auto-update in Portainer (5 min polling)

### Recommended Enhancements

**Week 1:**
- [ ] Import PostgreSQL dashboard (ID 9628)
- [ ] Set up email notifications
- [ ] Configure log retention policies
- [ ] Create backup strategy

**Week 2:**
- [ ] Add custom application metrics
- [ ] Create business dashboards
- [ ] Set up on-call rotation (PagerDuty, Opsgenie)
- [ ] Document runbooks for common issues

**Month 1:**
- [ ] Optimize disk usage and costs
- [ ] Tune scrape intervals for performance
- [ ] Add more hosts to monitoring
- [ ] Review and adjust retention policies

## Cost Breakdown

**DigitalOcean Setup (5 edge hosts):**
```
Central Server (4GB):         $24/month
Edge Agents:                  $0 (existing hosts)
VPC Transfer:                 $0 (free within region)
Remote VPS Egress:            ~$1-2/month
─────────────────────────────
Total:                        ~$25-26/month
```

**Compare to SaaS:**
- Datadog: ~$150-300/month (5 hosts)
- Grafana Cloud: ~$100-200/month
- **Savings: 75-85%**

## Common Issues

### "SSL Certificate Failed"
- **Wait**: DNS propagation takes 5-30 minutes
- **Check**: `dig monitoring.example.com` shows correct IP
- **Ports**: Ensure 80/443 are open (`sudo ufw allow 80,443/tcp`)

### "No Metrics from Edge Hosts"
- **Check logs**: `docker logs monitoring-prometheus`
- **VPC**: Verify hosts are in same VPC network
- **URL**: Confirm `CENTRAL_PROMETHEUS_URL` is correct
- **Test**: `curl http://10.116.0.2:9090/-/healthy` (VPC) or `curl https://monitoring.example.com` (HTTPS)

### "No Logs Appearing"
- **Check logs**: `docker logs monitoring-promtail`
- **Permissions**: Promtail needs access to `/var/run/docker.sock`
- **URL**: Confirm `CENTRAL_LOKI_URL` is correct

### "High CPU/Memory Usage"
- **Prometheus**: Reduce scrape intervals (30s instead of 15s)
- **Loki**: Reduce retention (60 days instead of 90)
- **Grafana**: Limit dashboard time range (1h default)

See [docs/TROUBLESHOOTING.md](./docs/TROUBLESHOOTING.md) for more solutions.

## Learning Resources

### Documentation
- [Architecture Deep Dive](./docs/ARCHITECTURE.md)
- [Complete Setup Guide](./docs/SETUP.md)
- [VPC Configuration](./docs/VPC-SETUP.md)
- [Git Submodules Setup](./docs/GIT-SETUP.md)

### Official Docs
- [Prometheus](https://prometheus.io/docs/)
- [Loki](https://grafana.com/docs/loki/)
- [Grafana](https://grafana.com/docs/grafana/)
- [Caddy](https://caddyserver.com/docs/)

### Community
- [Prometheus Slack](https://prometheus.io/community/)
- [Grafana Community](https://community.grafana.com/)
- [r/PrometheusMonitoring](https://reddit.com/r/PrometheusMonitoring)

## Support

- **Issues**: Open GitHub issue in relevant repository
- **Questions**: Check documentation first
- **Contributions**: Pull requests welcome!

## Quick Reference

### Central Server
- **Grafana**: `https://monitoring.example.com`
- **Prometheus** (VPC): `http://10.116.0.2:9090`
- **Loki** (VPC): `http://10.116.0.2:3100`

### Edge Hosts
- **Check status**: `docker ps | grep monitoring`
- **View logs**: `docker logs monitoring-<service>`
- **Restart**: `docker-compose restart`

### Useful Queries

**Prometheus:**
```promql
# All hosts up
up{job=~"cadvisor|node-exporter|postgres"}

# CPU usage per container
rate(container_cpu_usage_seconds_total[5m])

# Memory usage per host
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100

# PostgreSQL connections
pg_stat_database_numbackends
```

**Loki:**
```logql
# All logs from specific host
{host="edge-host-1"}

# Errors only
{job="docker"} |= "error" or "ERROR"

# Specific container
{container="my-app"} | json

# Rate of errors
rate({job="docker"} |= "error" [5m])
```

---

🎉 **Congratulations!** You now have a production-ready distributed monitoring stack.

For detailed configuration and advanced features, see the full documentation in `docs/`.
