# DigitalOcean VPC Setup Guide

VPC (Virtual Private Cloud) provides free, secure, private networking between your DigitalOcean droplets in the same region.

## Benefits

- **Zero egress cost**: Traffic between droplets in same VPC is free
- **Low latency**: Private network connection
- **Security**: No data traverses public internet
- **Simplified config**: No authentication needed between trusted hosts

## Prerequisites

- DigitalOcean account
- All droplets in the **same region** (e.g., FRA1)
- Droplets running Ubuntu 20.04+ or similar

## Step 1: Create VPC Network

### 1.1 Via DigitalOcean Console

1. Log in to DigitalOcean
2. Go to **Networking** → **VPC**
3. Click **Create VPC Network**
4. Configure:
   - **Name**: `monitoring-vpc`
   - **Region**: Your region (e.g., `Frankfurt 1`)
   - **IP range**: `10.116.0.0/20` (default is fine)
   - **Description**: `Private network for monitoring stack`
5. Click **Create VPC Network**

### 1.2 Via doctl CLI (Alternative)

```bash
# Install doctl
brew install doctl  # macOS
# or download from: https://github.com/digitalocean/doctl

# Authenticate
doctl auth init

# Create VPC
doctl vpcs create \
  --name monitoring-vpc \
  --region fra1 \
  --ip-range 10.116.0.0/20 \
  --description "Private network for monitoring stack"

# List VPCs to get the ID
doctl vpcs list
```

## Step 2: Assign Droplets to VPC

### 2.1 For Existing Droplets

**Important**: Adding a droplet to VPC requires reboot!

1. Go to **Droplets** → Select your droplet
2. Click **Networking** tab
3. In **VPC Networks** section, click **Edit**
4. Select `monitoring-vpc`
5. Click **Save** and **Reboot** when prompted

Repeat for:
- Central monitoring server
- All edge Docker hosts in the same region

### 2.2 For New Droplets

When creating a new droplet:
1. In creation wizard, go to **VPC Network** section
2. Select `monitoring-vpc`
3. Complete droplet creation

The droplet will be automatically connected to VPC.

### 2.3 Via doctl CLI

```bash
# Get droplet IDs
doctl compute droplet list

# Add droplet to VPC (requires reboot)
doctl compute droplet-action resize <droplet-id> \
  --resize-disk=false \
  --vpc-uuid <vpc-id>
```

## Step 3: Verify VPC Configuration

### 3.1 Check VPC IP Addresses

Each droplet gets a private IP in the VPC range (10.116.x.x).

```bash
# SSH into droplet
ssh root@<droplet-public-ip>

# List network interfaces
ip addr show

# Look for eth1 (VPC interface):
# eth1: <BROADCAST,MULTICAST,UP,LOWER_UP>
#     inet 10.116.0.2/20 scope global eth1
```

**Note the private IP** for each droplet:
- Central server: e.g., `10.116.0.2`
- Edge host 1: e.g., `10.116.0.3`
- Edge host 2: e.g., `10.116.0.4`

### 3.2 Test Connectivity

From edge host, ping central server via VPC:

```bash
# On edge host
ping 10.116.0.2

# Expected: replies, low latency (<1ms)
```

From central server, test Prometheus port:

```bash
# On central server, ensure Prometheus is bound to VPC IP
# (see Step 4)

# On edge host
curl http://10.116.0.2:9090/-/healthy

# Expected: "Prometheus is Healthy."
```

## Step 4: Configure Central Stack for VPC

### 4.1 Update docker-compose.yml

Edit `monitoring-grafana/docker-compose.yml`:

```yaml
services:
  prometheus:
    # ... existing config ...
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=2y'
      - '--web.listen-address=0.0.0.0:9090'  # Listen on all interfaces
      - '--web.enable-remote-write-receiver'
    # ... rest of config ...

  loki:
    # ... existing config ...
    # Already listens on 0.0.0.0:3100 by default
```

### 4.2 Update Portainer Environment Variables

In Portainer central stack, add:

```env
VPC_PRIVATE_IP=10.116.0.2
```

### 4.3 Optional: Bind to VPC IP Only (More Secure)

To prevent public access to Prometheus/Loki:

```yaml
  prometheus:
    command:
      - '--web.listen-address=10.116.0.2:9090'  # Only VPC IP
```

This means only VPC members can access Prometheus.

## Step 5: Configure Edge Stacks for VPC

### 5.1 Update Portainer Environment Variables

For each edge host in VPC, update stack environment:

```env
# Central server VPC private IP
CENTRAL_PROMETHEUS_URL=http://10.116.0.2:9090/api/v1/write
CENTRAL_LOKI_URL=http://10.116.0.2:3100/loki/api/v1/push

# This host's identifier
HOSTNAME=edge-host-1

# NO authentication needed for VPC!
# Remove BASIC_AUTH_USER and BASIC_AUTH_PASSWORD
```

### 5.2 Redeploy Edge Stacks

In Portainer:
1. Go to **Stacks** → `monitoring-edge`
2. Click **Pull and redeploy**

Or via Git (if GitOps enabled):
```bash
# Commit environment variable changes
git commit -am "Switch to VPC networking"
git push

# Portainer will auto-deploy
```

## Step 6: Verify VPC Monitoring

### 6.1 Check Edge Prometheus Logs

```bash
# On edge host
docker logs monitoring-prometheus

# Should see successful remote writes:
# level=info msg="Remote write succeeded" url=http://10.116.0.2:9090/api/v1/write
```

### 6.2 Check Promtail Logs

```bash
# On edge host
docker logs monitoring-promtail

# Should see successful pushes:
# level=info msg="Successfully sent batch" ...
```

### 6.3 Verify in Grafana

1. Open Grafana: `https://monitoring.example.com`
2. Go to **Explore** → **Prometheus**
3. Query:
   ```promql
   up{job="cadvisor"}
   ```
4. Should see metrics from all VPC hosts

### 6.4 Check Network Traffic

VPC traffic should be free:

```bash
# On central server, monitor network traffic
sudo apt-get install iftop
sudo iftop -i eth1  # eth1 is VPC interface

# Should see connections from VPC IPs (10.116.x.x)
```

Verify in DigitalOcean billing that egress is not charged for VPC traffic.

## Mixed Setup: VPC + Remote VPS

You can have both VPC hosts and remote VPS in the same monitoring stack.

### Central Stack Configuration

```yaml
# Caddy configuration supports both:
# - VPC hosts connect directly to Prometheus/Loki
# - Remote VPS connect via Caddy proxy with Basic Auth
```

### Remote VPS Configuration

Remote VPS outside VPC still uses HTTPS + Basic Auth:

```env
CENTRAL_PROMETHEUS_URL=https://monitoring.example.com/prometheus/api/v1/write
CENTRAL_LOKI_URL=https://monitoring.example.com/loki/api/v1/push
HOSTNAME=remote-vps-1
BASIC_AUTH_USER=remote
BASIC_AUTH_PASSWORD=your-password
```

## Firewall Configuration

### DigitalOcean Cloud Firewall

Create firewall rules for VPC:

1. Go to **Networking** → **Firewalls**
2. Click **Create Firewall**
3. Configure:

**Inbound Rules** (for central server):
```
Type        Protocol  Port Range  Sources
SSH         TCP       22          Your IP
HTTPS       TCP       443         All IPv4
Prometheus  TCP       9090        VPC: monitoring-vpc
Loki        TCP       3100        VPC: monitoring-vpc
```

**Outbound Rules**:
```
Type        Protocol  Port Range  Destinations
All         All       All         All IPv4
```

4. **Apply to Droplets**: Select central server
5. Click **Create Firewall**

### UFW (On-Host Firewall)

Alternatively, configure UFW on each droplet:

```bash
# On central server
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 443/tcp   # HTTPS (Grafana via Caddy)
sudo ufw allow from 10.116.0.0/20 to any port 9090  # Prometheus from VPC
sudo ufw allow from 10.116.0.0/20 to any port 3100  # Loki from VPC
sudo ufw enable

# On edge hosts (default deny is fine)
sudo ufw allow 22/tcp    # SSH only
sudo ufw enable
```

## Troubleshooting

### VPC Interface Not Present

```bash
# Check interfaces
ip addr show

# If eth1 (VPC) is missing:
# 1. Verify droplet is assigned to VPC in DO console
# 2. Reboot droplet
sudo reboot
```

### Cannot Reach Central Server from Edge

```bash
# On edge host, check route
ip route show

# Should have route for 10.116.0.0/20
# 10.116.0.0/20 dev eth1 proto kernel scope link src 10.116.0.3

# Test connectivity
ping 10.116.0.2
telnet 10.116.0.2 9090

# If fails, check firewall
sudo ufw status
```

### Prometheus Remote Write Fails

```bash
# Check Prometheus is listening on VPC IP
# On central server
docker exec monitoring-prometheus netstat -tlnp | grep 9090

# Should show: 0.0.0.0:9090 or 10.116.0.2:9090

# Test from edge host
curl http://10.116.0.2:9090/-/healthy

# If fails:
# - Check firewall rules
# - Verify Prometheus command in docker-compose.yml
# - Check Docker network mode (should be bridge or host)
```

### High VPC Latency

```bash
# Check VPC performance
ping -c 100 10.116.0.2

# Expected: <1ms average
# If >5ms: Check if droplets are in same region

# Verify region
doctl compute droplet list --format Name,Region
```

### VPC Traffic Being Charged

If you see egress charges for VPC traffic:

1. Verify traffic is using private IPs (10.116.x.x)
2. Check DigitalOcean networking graphs (should show internal traffic)
3. Contact DigitalOcean support if charged incorrectly

VPC traffic within same region is **always free** as of 2024.

## Cost Savings

Example for 5 droplets, 10GB/day monitoring traffic:

**Without VPC** (using public IPs):
- Egress: 10GB/day × 30 days × $0.01/GB = $3/month
- For 5 hosts: $15/month

**With VPC**:
- Egress: $0/month

**Savings**: $15/month = $180/year

## Best Practices

1. **Use VPC for all DO hosts in same region**: No reason not to
2. **Bind services to VPC IP only**: More secure (not exposed publicly)
3. **Separate VPCs for different environments**: prod-vpc, staging-vpc
4. **Document VPC IPs**: Keep a list of which droplet has which VPC IP
5. **Test connectivity after any changes**: Always verify with ping/curl

## Further Reading

- [DigitalOcean VPC Documentation](https://docs.digitalocean.com/products/networking/vpc/)
- [VPC Peering](https://docs.digitalocean.com/products/networking/vpc/how-to/peering/) (for cross-region)
- [Cloud Firewalls](https://docs.digitalocean.com/products/networking/firewalls/)
