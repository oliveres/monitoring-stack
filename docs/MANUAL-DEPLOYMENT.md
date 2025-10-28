# Manual Deployment Guide (Without Portainer)

Complete guide for deploying the monitoring stack using Docker Compose CLI without Portainer.

## Overview

This guide is for users who prefer command-line deployment over Portainer's GUI. The setup is **fully persistent** - containers will automatically restart after:
- Container crash
- Docker daemon restart
- Server reboot

## Prerequisites

- Docker 20.10+
- Docker Compose 2.0+
- Git installed
- Root or sudo access
- Domain name pointing to central server (for SSL)

## Persistence Guarantee

All `docker-compose.yml` files in this project use:

```yaml
services:
  service-name:
    restart: unless-stopped  # ← This ensures auto-start
```

**What `restart: unless-stopped` means:**
- ✅ Container restarts after crash
- ✅ Container starts after Docker daemon restart
- ✅ Container starts after server reboot
- ❌ Container does NOT start only if manually stopped with `docker stop`

## Deployment Commands

### Critical: Use Detached Mode

```bash
# ❌ WRONG - stops when terminal closes
docker compose up

# ✅ CORRECT - runs in background
docker compose up -d
```

## Central Server Deployment

### Step 1: Prepare Server

```bash
# Install Docker (if not already installed)
curl -fsSL https://get.docker.com | sh

# Ensure Docker starts on boot
sudo systemctl enable docker
sudo systemctl start docker

# Verify Docker is enabled
systemctl is-enabled docker
# Should return: enabled
```

### Step 2: Configure DNS

Before deployment, ensure DNS is configured:

```bash
# Check DNS resolution
dig monitoring.example.com +short

# Should return your server's public IP
# Wait 5-30 minutes for DNS propagation if just configured
```

### Step 3: Clone and Configure

```bash
# Clone central stack repository
cd /root  # or your preferred location
git clone https://github.com/oliveres/monitoring-grafana.git
cd monitoring-grafana

# Create environment file
cp .env.example .env

# Edit configuration
nano .env
```

**Required environment variables:**

```env
# Domain for SSL certificate (required)
DOMAIN=monitoring.example.com

# Grafana admin credentials
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=your-secure-password-here

# Prometheus retention (default: 2y)
PROMETHEUS_RETENTION=2y

# For remote VPS access (optional)
BASIC_AUTH_USER=remote
BASIC_AUTH_PASSWORD_HASH=$2a$12$...your-bcrypt-hash...
```

**Generate Basic Auth hash:**

```bash
# Install htpasswd
sudo apt-get install apache2-utils

# Generate bcrypt hash
htpasswd -nbB remote yourpassword

# Output example: remote:$2y$05$bQlNNr5rxpTLAY5DouzSL...
# Copy only the hash part (after "remote:")

# IMPORTANT: In .env file, escape $ signs by doubling them ($$)
# Example:
# From htpasswd: $2y$05$bQlNNr...
# In .env:       $$2y$$05$$bQlNNr...
```

### Step 4: Deploy Stack

```bash
# Deploy in detached mode
docker compose up -d

# Verify all containers are running
docker compose ps

# Expected output:
# NAME                      STATUS
# monitoring-caddy          Up
# monitoring-grafana        Up
# monitoring-prometheus     Up
# monitoring-loki           Up
# monitoring-promtail       Up
```

### Step 5: Verify Deployment

```bash
# Check container logs
docker compose logs -f caddy
# Watch for SSL certificate success

docker compose logs -f grafana
# Should show: "HTTP Server Listen"

# Verify restart policy
docker inspect monitoring-grafana | grep -A 5 RestartPolicy
# Should show: "Name": "unless-stopped"

# Access Grafana
curl -I https://monitoring.example.com
# Should return: 200 OK
```

### Step 6: Open Firewall Ports

```bash
# Allow HTTP/HTTPS for Caddy SSL
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Allow SSH (if not already)
sudo ufw allow 22/tcp

# Enable firewall
sudo ufw enable

# Verify rules
sudo ufw status
```

### Step 7: Verify Grafana

1. Open browser: `https://monitoring.example.com`
2. Login with credentials from `.env`
3. Navigate to: Configuration → Data sources
4. Verify both Prometheus and Loki are green (healthy)

## Edge Host Deployment

### For Standard Docker Hosts (Basic)

```bash
# Clone repository
cd /root
git clone https://github.com/oliveres/monitoring-edge-basic.git
cd monitoring-edge-basic

# Create environment file
cp .env.example .env
nano .env
```

**For DigitalOcean VPC setup:**

```env
HOSTNAME=edge-host-1
CENTRAL_PROMETHEUS_URL=http://10.116.0.2:9090/api/v1/write
CENTRAL_LOKI_URL=http://10.116.0.2:3100/loki/api/v1/push
# No BASIC_AUTH_* needed for VPC
```

**For remote VPS (HTTPS):**

```env
HOSTNAME=remote-vps-1
CENTRAL_PROMETHEUS_URL=https://monitoring.example.com/prometheus/api/v1/write
CENTRAL_LOKI_URL=https://monitoring.example.com/loki/api/v1/push
BASIC_AUTH_USER=remote
BASIC_AUTH_PASSWORD=yourpassword
```

```bash
# Deploy
docker compose up -d

# Verify
docker compose ps
docker compose logs -f prometheus | grep "Remote write succeeded"
docker compose logs -f promtail | grep "Successfully sent"
```

### For PostgreSQL Hosts

```bash
# Clone PostgreSQL-enabled stack
cd /root
git clone https://github.com/oliveres/monitoring-edge-postgres.git
cd monitoring-edge-postgres

# Ensure dispatch-network exists (for PostgreSQL connection)
docker network create dispatch-network

# Create and configure .env
cp .env.example .env
nano .env
```

**Additional required variables:**

```env
# PostgreSQL connection string
POSTGRES_DATA_SOURCE_NAME=postgresql://user:password@host:5432/dbname?sslmode=disable

# Plus all variables from basic edge setup (HOSTNAME, CENTRAL_*_URL, etc.)
```

```bash
# Deploy
docker compose up -d

# Verify PostgreSQL exporter
docker logs monitoring-postgres-exporter
# Should see: "Listening on :9187"

# Test PostgreSQL connection
docker exec monitoring-postgres-exporter \
  psql "$POSTGRES_DATA_SOURCE_NAME" -c "SELECT version();"
```

## Management Commands

### Viewing Logs

```bash
# All containers
docker compose logs -f

# Specific service
docker compose logs -f grafana
docker compose logs -f prometheus

# Last 50 lines
docker compose logs --tail 50

# Since specific time
docker compose logs --since 1h
```

### Restarting Services

```bash
# Restart all services
docker compose restart

# Restart specific service
docker compose restart grafana
docker compose restart prometheus

# Full restart (recreate containers)
docker compose down
docker compose up -d
```

### Updating from Git

```bash
cd /root/monitoring-grafana  # or edge stack directory

# Pull latest changes
git pull

# Apply changes (recreates containers if needed)
docker compose up -d

# View what changed
git log -1 --stat
```

### Stopping Stack

```bash
# Stop all containers (temporary)
docker compose stop

# Start again
docker compose start

# Stop and remove containers (volumes preserved)
docker compose down

# Stop and remove EVERYTHING including data (⚠️ DANGEROUS!)
docker compose down -v
```

### Rebuilding After Dockerfile Changes

```bash
# Rebuild images
docker compose build

# Rebuild and restart
docker compose up -d --build

# Force rebuild (no cache)
docker compose build --no-cache
docker compose up -d
```

### Checking Resource Usage

```bash
# Real-time stats
docker stats

# Disk usage by volumes
docker system df -v

# Specific volume size
du -sh /var/lib/docker/volumes/monitoring-grafana_prometheus-data
du -sh /var/lib/docker/volumes/monitoring-grafana_loki-data
```

## Persistence Testing

### Test Automatic Restart

```bash
# 1. Deploy stack
docker compose up -d

# 2. Verify containers are running
docker compose ps

# 3. Restart Docker daemon
sudo systemctl restart docker

# 4. Wait a few seconds
sleep 5

# 5. Check containers are back up
docker compose ps
# All containers should be running again
```

### Test Server Reboot

```bash
# 1. Note running containers
docker ps

# 2. Reboot server
sudo reboot

# 3. SSH back in after reboot

# 4. Check Docker is running
systemctl status docker

# 5. Check containers are running
docker ps
# All containers should have restarted automatically
```

## GitOps-like Auto-Update (Optional)

If you want automatic updates from Git without Portainer:

### Create Systemd Service

```bash
# Create service file
sudo nano /etc/systemd/system/monitoring-update.service
```

```ini
[Unit]
Description=Update monitoring stack from Git
After=network.target docker.service
Requires=docker.service

[Service]
Type=oneshot
WorkingDirectory=/root/monitoring-grafana
ExecStart=/bin/bash -c 'git pull && docker compose up -d'
User=root

[Install]
WantedBy=multi-user.target
```

### Create Systemd Timer

```bash
# Create timer file
sudo nano /etc/systemd/system/monitoring-update.timer
```

```ini
[Unit]
Description=Update monitoring stack every 5 minutes

[Timer]
OnBootSec=5min
OnUnitActiveSec=5min
Persistent=true

[Install]
WantedBy=timers.target
```

### Enable Auto-Update

```bash
# Reload systemd
sudo systemctl daemon-reload

# Enable and start timer
sudo systemctl enable monitoring-update.timer
sudo systemctl start monitoring-update.timer

# Check timer status
sudo systemctl status monitoring-update.timer

# View timer list
systemctl list-timers | grep monitoring

# Manually trigger update
sudo systemctl start monitoring-update.service

# View logs
journalctl -u monitoring-update.service -f
```

## Docker Daemon Configuration

### Configure Log Rotation

```bash
# Edit Docker daemon config
sudo nano /etc/docker/daemon.json
```

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

```bash
# Restart Docker to apply
sudo systemctl restart docker
```

### Verify Docker Auto-Start

```bash
# Check Docker service is enabled
systemctl is-enabled docker
# Should return: enabled

# If not enabled, enable it
sudo systemctl enable docker

# Check Docker service status
systemctl status docker
# Should show: active (running)
```

## Backup and Restore

### Backup Volumes

```bash
# Stop stack
cd /root/monitoring-grafana
docker compose down

# Backup volumes
sudo tar -czf prometheus-backup-$(date +%Y%m%d).tar.gz \
  /var/lib/docker/volumes/monitoring-grafana_prometheus-data

sudo tar -czf loki-backup-$(date +%Y%m%d).tar.gz \
  /var/lib/docker/volumes/monitoring-grafana_loki-data

sudo tar -czf grafana-backup-$(date +%Y%m%d).tar.gz \
  /var/lib/docker/volumes/monitoring-grafana_grafana-data

# Restart stack
docker compose up -d

# Move backups to safe location
mv *-backup-*.tar.gz /backup/
```

### Restore Volumes

```bash
# Stop stack
docker compose down

# Remove old volumes
docker volume rm monitoring-grafana_prometheus-data
docker volume rm monitoring-grafana_loki-data
docker volume rm monitoring-grafana_grafana-data

# Extract backups
sudo tar -xzf prometheus-backup-20250128.tar.gz -C /
sudo tar -xzf loki-backup-20250128.tar.gz -C /
sudo tar -xzf grafana-backup-20250128.tar.gz -C /

# Start stack
docker compose up -d
```

## Troubleshooting

### Containers Not Starting After Reboot

```bash
# Check Docker daemon
systemctl status docker

# If not running, start it
sudo systemctl start docker

# Check container restart policy
docker inspect monitoring-grafana | grep RestartPolicy

# Should show:
# "RestartPolicy": {
#     "Name": "unless-stopped",
# }

# If wrong policy, recreate with correct docker-compose.yml
cd /root/monitoring-grafana
docker compose down
docker compose up -d
```

### SSL Certificate Issues

```bash
# Check Caddy logs
docker compose logs caddy

# Common issues:
# 1. DNS not propagated - wait longer
# 2. Ports 80/443 not open - check firewall
# 3. Rate limit - wait 1 hour or use staging

# Test DNS
dig monitoring.example.com +short

# Test ports
sudo netstat -tlnp | grep -E ':(80|443)'

# Restart Caddy to retry SSL
docker compose restart caddy
```

### Prometheus Not Receiving Data

```bash
# On edge host, check logs
docker compose logs prometheus | grep -i error

# Test connectivity
# VPC:
curl http://10.116.0.2:9090/-/healthy
# HTTPS:
curl https://monitoring.example.com/prometheus/api/v1/write

# Check environment variables
docker compose config | grep CENTRAL_PROMETHEUS_URL
```

### Disk Space Issues

```bash
# Check disk usage
df -h

# Check Docker volumes
docker system df -v

# Clean up old containers/images
docker system prune -a

# Reduce Prometheus retention
nano .env
# Set: PROMETHEUS_RETENTION=1y
docker compose up -d

# Reduce Loki retention
nano loki/loki-config.yaml
# Set: retention_period: 60d
docker compose restart loki
```

## Security Best Practices

1. **Use strong passwords** in `.env` files
2. **Set correct file permissions**:
   ```bash
   chmod 600 .env
   chown root:root .env
   ```
3. **Keep software updated**:
   ```bash
   git pull
   docker compose pull
   docker compose up -d
   ```
4. **Use firewall** (ufw or iptables)
5. **Monitor logs** for suspicious activity
6. **Use VPC** when possible (free and secure on DigitalOcean)
7. **Regular backups** of volumes

## Comparison: Portainer vs. Manual

| Feature | Portainer | Docker Compose CLI |
|---------|-----------|-------------------|
| Auto-start after reboot | ✅ Yes | ✅ Yes (with `restart: unless-stopped`) |
| GitOps auto-update | ✅ Yes (built-in) | ✅ Yes (with systemd timer) |
| Web UI | ✅ Yes | ❌ No (CLI only) |
| Environment variables | ✅ GUI editor | ✅ .env file |
| Multi-server management | ✅ Single UI | ❌ SSH to each server |
| Resource overhead | ~100MB RAM | ~0MB (no overhead) |
| Learning curve | Easy (GUI) | Moderate (CLI) |
| Flexibility | Medium | High (full control) |

## When to Use Manual Deployment

**Use Docker Compose CLI if:**
- ✅ You prefer command-line over GUI
- ✅ You have 1-3 servers only
- ✅ You want minimal overhead
- ✅ You need full control over deployment
- ✅ You're comfortable with SSH and Bash

**Use Portainer if:**
- ✅ You manage many servers (5+)
- ✅ You prefer GUI
- ✅ You want built-in GitOps
- ✅ Non-technical users need access
- ✅ You want centralized management

## Getting Help

- Check [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) for common issues
- View logs: `docker compose logs -f`
- Check Docker status: `systemctl status docker`
- Test connectivity: `curl` and `ping`
- GitHub Issues: Report problems in respective repositories

## Quick Reference

### Daily Commands

```bash
# Check status
docker compose ps

# View logs
docker compose logs -f

# Restart service
docker compose restart grafana

# Update from Git
git pull && docker compose up -d
```

### Troubleshooting

```bash
# Check Docker
systemctl status docker

# Check restart policy
docker inspect <container> | grep RestartPolicy

# Check disk space
df -h
docker system df

# Clean up
docker system prune
```

### Emergency

```bash
# Stop everything
docker compose down

# Full reset (⚠️ deletes data!)
docker compose down -v

# Start fresh
docker compose up -d
```

---

**Summary:** Manual deployment with `docker compose up -d` and `restart: unless-stopped` (already configured in all stacks) provides full persistence. Containers will automatically restart after any server or Docker daemon restart. Portainer adds convenience but is not required for persistence.
