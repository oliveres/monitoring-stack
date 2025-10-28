# Troubleshooting Guide

Common issues and solutions for the distributed monitoring stack.

## General Debugging

### Check Container Status

```bash
# List all monitoring containers
docker ps -a | grep monitoring

# Check specific container logs
docker logs monitoring-<container-name>

# Follow logs in real-time
docker logs -f monitoring-<container-name> --tail 50

# Check container resource usage
docker stats
```

### Restart Stack

```bash
# Via Portainer
# Stacks → Your stack → Stop → Start

# Via Docker CLI
docker-compose -f /path/to/docker-compose.yml restart

# Restart specific service
docker-compose -f /path/to/docker-compose.yml restart prometheus
```

## Central Stack Issues

### 1. Caddy SSL Certificate Fails

**Symptoms:**
- Cannot access Grafana via HTTPS
- Browser shows certificate error
- Caddy logs show "failed to obtain certificate"

**Common Causes:**

**a) DNS not pointing to server**
```bash
# Check DNS resolution
dig monitoring.example.com +short

# Should return your server's public IP
# If not, update DNS and wait for propagation (can take hours)
```

**b) Port 80/443 not open**
```bash
# Check firewall
sudo ufw status

# Open ports if needed
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Check DigitalOcean firewall in console
# Ensure inbound HTTPS (443) is allowed
```

**c) Rate limit from Let's Encrypt**
```bash
# Check Caddy logs
docker logs monitoring-caddy | grep "rate limit"

# Wait 1 hour or use staging environment temporarily
# Edit Caddyfile to add: staging: true
```

**Solution:**
```bash
# Verify all prerequisites
# 1. DNS points to server ✓
# 2. Ports 80/443 open ✓
# 3. No rate limits ✓

# Remove Caddy data and retry
docker-compose down
sudo rm -rf caddy/data/*
docker-compose up -d

# Watch logs
docker logs -f monitoring-caddy
```

### 2. Prometheus Not Receiving Remote Write

**Symptoms:**
- Edge hosts not showing up in Prometheus
- Grafana shows "No data"
- Edge logs: "connection refused" or "401 unauthorized"

**Diagnosis:**
```bash
# On central server, check Prometheus is listening
docker exec monitoring-prometheus netstat -tlnp | grep 9090

# Test remote write endpoint
curl -X POST http://localhost:9090/api/v1/write

# Check Prometheus config
docker exec monitoring-prometheus cat /etc/prometheus/prometheus.yml | grep remote_write_receiver
```

**Solutions:**

**a) Remote write receiver not enabled**
```yaml
# In prometheus command:
- '--web.enable-remote-write-receiver'  # Must be present
```

**b) Firewall blocking**
```bash
# For VPC setup
sudo ufw allow from 10.116.0.0/20 to any port 9090

# For HTTPS setup (via Caddy proxy)
# Ensure Caddy is running and proxying correctly
```

**c) Wrong URL in edge config**
```env
# VPC setup
CENTRAL_PROMETHEUS_URL=http://10.116.0.2:9090/api/v1/write

# HTTPS setup
CENTRAL_PROMETHEUS_URL=https://monitoring.example.com/prometheus/api/v1/write
```

### 3. Loki Not Receiving Logs

**Symptoms:**
- No logs in Grafana Explore
- Promtail logs: "connection refused" or "server returned HTTP status 4XX"

**Diagnosis:**
```bash
# Check Loki is running
docker logs monitoring-loki

# Test Loki endpoint
curl http://localhost:3100/ready

# Check Loki config
docker exec monitoring-loki cat /etc/loki/loki-config.yaml
```

**Solutions:**

**a) Loki not listening on correct interface**
```yaml
# In loki-config.yaml:
server:
  http_listen_address: 0.0.0.0  # Not 127.0.0.1
  http_listen_port: 3100
```

**b) Wrong push URL**
```env
# Correct formats:
CENTRAL_LOKI_URL=http://10.116.0.2:3100/loki/api/v1/push    # VPC
CENTRAL_LOKI_URL=https://monitoring.example.com/loki/api/v1/push  # HTTPS
```

**c) Authentication issues (HTTPS setup)**
```bash
# Test with Basic Auth
curl -u remote:password https://monitoring.example.com/loki/api/v1/push

# Check Caddy Basic Auth config
docker exec monitoring-caddy cat /etc/caddy/Caddyfile
```

### 4. Grafana Can't Connect to Datasources

**Symptoms:**
- Datasources show red (unhealthy)
- "Bad Gateway" or "Connection refused" errors

**Diagnosis:**
```bash
# Check all containers are on same network
docker network inspect monitoring-grafana_monitoring

# Check datasource URLs in Grafana
# Grafana → Configuration → Data sources
```

**Solution:**
```yaml
# Datasource URLs should use Docker service names:
# Prometheus: http://prometheus:9090
# Loki: http://loki:3100

# NOT external IPs or localhost!
```

## Edge Stack Issues

### 1. Prometheus Agent Not Sending Data

**Symptoms:**
- Metrics not appearing in central Prometheus
- Edge logs: "remote write failed"

**Diagnosis:**
```bash
# Check Prometheus logs
docker logs monitoring-prometheus

# Look for errors like:
# - "connection refused"
# - "context deadline exceeded"
# - "401 Unauthorized"
# - "certificate error"
```

**Solutions:**

**a) Wrong central URL**
```bash
# Verify environment variable
docker exec monitoring-prometheus env | grep CENTRAL

# Test connectivity manually
# VPC:
curl -v http://10.116.0.2:9090/-/healthy
# HTTPS:
curl -v https://monitoring.example.com/prometheus/api/v1/write
```

**b) Network connectivity**
```bash
# VPC setup - test private network
ping 10.116.0.2
telnet 10.116.0.2 9090

# HTTPS setup - test public endpoint
curl -I https://monitoring.example.com

# Check DNS
nslookup monitoring.example.com
```

**c) Authentication (HTTPS setup)**
```bash
# Verify Basic Auth credentials
curl -u remote:password https://monitoring.example.com/prometheus/api/v1/write

# Check environment variables
docker exec monitoring-prometheus env | grep BASIC_AUTH
```

**d) Certificate errors (HTTPS setup)**
```bash
# Check if SSL is valid
openssl s_client -connect monitoring.example.com:443

# If self-signed cert, add to Prometheus:
# (Not recommended, fix SSL instead)
--web.remote-write.tls-config.insecure-skip-verify
```

### 2. Promtail Can't Read Docker Logs

**Symptoms:**
- No logs appearing in Loki
- Promtail logs: "permission denied" or "no such file"

**Diagnosis:**
```bash
# Check Promtail logs
docker logs monitoring-promtail

# Verify Docker socket access
docker exec monitoring-promtail ls -l /var/run/docker.sock

# Check log directory
docker exec monitoring-promtail ls /var/lib/docker/containers
```

**Solutions:**

**a) Docker socket permissions**
```yaml
# In docker-compose.yml, Promtail needs:
user: "0:0"  # Run as root
volumes:
  - /var/run/docker.sock:/var/run/docker.sock:ro  # :ro = read-only
  - /var/lib/docker/containers:/var/lib/docker/containers:ro
```

**b) SELinux issues (CentOS/RHEL)**
```bash
# Check SELinux status
sestatus

# Temporarily disable for testing
sudo setenforce 0

# Permanent fix: Add SELinux context
chcon -Rt svirt_sandbox_file_t /var/lib/docker/containers
```

**c) Wrong log path**
```bash
# Verify Docker log location
docker info | grep "Docker Root Dir"

# Common paths:
# Ubuntu/Debian: /var/lib/docker
# Custom: Check Docker daemon.json
```

### 3. cAdvisor Not Collecting Metrics

**Symptoms:**
- No container metrics in Prometheus
- Query `up{job="cadvisor"}` returns no results
- cAdvisor container keeps restarting

**Diagnosis:**
```bash
# Check cAdvisor logs
docker logs monitoring-cadvisor

# Common errors:
# - "failed to get cgroup stats"
# - "no such file or directory"
```

**Solutions:**

**a) cgroups not enabled (Raspberry Pi, older systems)**
```bash
# Check kernel command line
cat /proc/cmdline | grep cgroup

# If missing, edit grub config
sudo nano /etc/default/grub

# Add to GRUB_CMDLINE_LINUX:
cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1 swapaccount=1

# Update grub
sudo update-grub
sudo reboot
```

**b) Missing privileged mode**
```yaml
# cAdvisor requires:
privileged: true
devices:
  - /dev/kmsg
```

**c) Wrong volume mounts**
```yaml
volumes:
  - /:/rootfs:ro
  - /var/run:/var/run:rw
  - /sys:/sys:ro
  - /var/lib/docker/:/var/lib/docker:ro
  - /dev/disk/:/dev/disk:ro
  - /etc/machine-id:/etc/machine-id:ro
```

### 4. PostgreSQL Exporter Connection Failed

**Symptoms:**
- PostgreSQL metrics missing
- Exporter logs: "connection refused" or "authentication failed"

**Diagnosis:**
```bash
# Check exporter logs
docker logs monitoring-postgres-exporter

# Verify connection string
docker exec monitoring-postgres-exporter env | grep DATA_SOURCE_NAME

# Test connection manually
docker exec monitoring-postgres-exporter \
  psql "$DATA_SOURCE_NAME" -c "SELECT version();"
```

**Solutions:**

**a) Wrong connection string format**
```env
# Correct format:
DATA_SOURCE_NAME=postgresql://user:password@host:5432/database?sslmode=disable

# Common mistakes:
# - Missing sslmode (should be disable unless you have SSL)
# - Wrong host (should be accessible from container network)
# - Wrong port (default PostgreSQL is 5432)
```

**b) Network not connected**
```yaml
# Postgres exporter needs TWO networks:
networks:
  - internal      # For Prometheus to scrape
  - dispatch-network  # External network to reach PostgreSQL
```

**c) PostgreSQL not allowing connections**
```bash
# Check PostgreSQL logs
docker logs <postgres-container>

# Check pg_hba.conf allows connections from Docker network
# Edit postgresql.conf:
listen_addresses = '*'

# Edit pg_hba.conf:
host all all 172.16.0.0/12 md5  # Docker default network range
```

## Network Issues

### VPC Connectivity Problems

**Symptoms:**
- Edge hosts can't reach central server via VPC
- High latency (>10ms)

**Solutions:**

```bash
# 1. Verify VPC membership
# DigitalOcean console: Networking → VPC → Check all droplets listed

# 2. Check VPC interface exists
ip addr show eth1

# 3. Test connectivity
ping 10.116.0.2

# 4. Check routing
ip route show | grep 10.116

# 5. Verify droplets are in SAME REGION
# VPC only works within one region!

# 6. Reboot if interface missing
sudo reboot
```

### DNS Resolution Issues

**Symptoms:**
- Edge hosts can't resolve central domain
- "no such host" errors

**Solutions:**

```bash
# Test DNS
nslookup monitoring.example.com
dig monitoring.example.com

# Try different DNS servers
sudo nano /etc/resolv.conf
# Add: nameserver 8.8.8.8

# Restart networking
sudo systemctl restart systemd-resolved

# Alternative: Use IP instead of domain
CENTRAL_PROMETHEUS_URL=https://203.0.113.10/prometheus/api/v1/write
```

## Performance Issues

### High Prometheus Memory Usage

**Symptoms:**
- Prometheus container using >4GB RAM
- OOM (Out of Memory) errors

**Solutions:**

```bash
# 1. Check retention time
docker exec monitoring-prometheus cat /etc/prometheus/prometheus.yml

# Reduce if needed (central server):
--storage.tsdb.retention.time=1y  # Instead of 2y

# 2. Check scrape intervals
# Reduce frequency in prometheus.yml:
scrape_interval: 30s  # Instead of 15s

# 3. Increase server RAM
# Central server should have minimum 4GB, recommend 8GB

# 4. Enable Prometheus downsampling (future enhancement)
```

### Loki Disk Usage Too High

**Symptoms:**
- Loki data directory growing rapidly
- Disk space warnings

**Solutions:**

```bash
# 1. Check current usage
du -sh /var/lib/docker/volumes/*_loki-data

# 2. Reduce retention in loki-config.yaml:
limits_config:
  retention_period: 60d  # Instead of 90d

# 3. Increase compaction frequency
chunk_store_config:
  max_look_back_period: 30d

# 4. Filter noisy logs in Promtail config:
pipeline_stages:
  - match:
      selector: '{job="docker"}'
      stages:
        - drop:
            expression: ".*healthcheck.*"  # Drop healthcheck logs
```

### Slow Grafana Dashboards

**Symptoms:**
- Dashboards take >10s to load
- "Query timeout" errors

**Solutions:**

```bash
# 1. Reduce query time range
# Use 1h or 6h instead of 7d in dashboards

# 2. Increase Prometheus query timeout
# In prometheus.yml:
--query.timeout=2m

# 3. Use recording rules for expensive queries
# Create prometheus/rules.yml with pre-computed metrics

# 4. Optimize Grafana queries
# Use rate() instead of irate() for long ranges
# Add more specific label filters
```

## Data Issues

### Gaps in Metrics Data

**Symptoms:**
- Missing data points in Grafana graphs
- Edge logs show temporary connection failures

**Common Causes:**

1. **Network hiccup** - Normal, data will resume
2. **Prometheus restart** - Brief gap is expected
3. **Remote write queue full** - Check edge Prometheus memory

**Solutions:**

```bash
# Check for sustained failures
docker logs monitoring-prometheus | grep "remote write failed" | tail -20

# If persistent, check:
# 1. Network stability
# 2. Central server capacity
# 3. Edge agent memory limits

# Increase remote write queue size (edge config):
--storage.remote.write.queue.max-samples-per-send=10000
```

### Duplicate Metrics

**Symptoms:**
- Multiple time series for same metric
- Inconsistent values in Grafana

**Causes:**

1. **Multiple edge agents with same hostname**
2. **Host redeployed with different instance ID**

**Solutions:**

```bash
# Ensure unique HOSTNAME for each edge host
docker exec monitoring-prometheus env | grep HOSTNAME

# Use instance label to identify:
up{job="cadvisor", instance="edge-host-1"}
```

## Getting Help

### Collect Diagnostic Information

```bash
#!/bin/bash
# Save as debug-info.sh

echo "=== Docker Version ==="
docker --version
docker-compose --version

echo "=== Container Status ==="
docker ps -a | grep monitoring

echo "=== Container Logs (last 50 lines) ==="
for container in $(docker ps --filter "name=monitoring" --format "{{.Names}}"); do
  echo "--- $container ---"
  docker logs --tail 50 $container
done

echo "=== Network Info ==="
ip addr show
ip route show

echo "=== Disk Usage ==="
df -h
du -sh /var/lib/docker/volumes/*monitoring*

echo "=== System Resources ==="
free -h
docker stats --no-stream
```

Run and save output:
```bash
chmod +x debug-info.sh
./debug-info.sh > debug-output.txt
```

### Where to Get Help

1. **GitHub Issues**: Post in respective repository
2. **Include**: debug-output.txt, docker-compose.yml (without secrets)
3. **Check**: Existing issues first
4. **Prometheus**: https://prometheus.io/community/
5. **Loki**: https://grafana.com/docs/loki/latest/
6. **Grafana**: https://community.grafana.com/
