# Git Submodules Setup Guide

This guide explains how to set up the Git repositories and submodules for the distributed monitoring stack.

## Overview

The monitoring stack uses **4 GitHub repositories**:

1. **monitoring-stack** (umbrella repo) - Documentation and submodule references
2. **monitoring-grafana** - Central server stack
3. **monitoring-edge-basic** - Edge agent without PostgreSQL
4. **monitoring-edge-postgres** - Edge agent with PostgreSQL monitoring

## Step 1: Create GitHub Repositories

### 1.1 Create Repositories on GitHub

Go to GitHub and create **4 new repositories**:

1. `monitoring-stack` (Public or Private)
   - Description: "Distributed monitoring infrastructure with Prometheus, Loki, and Grafana"
   - Initialize: No (we'll push existing code)

2. `monitoring-grafana` (Private recommended)
   - Description: "Central monitoring server with Grafana, Prometheus, Loki, and Caddy"
   - Initialize: No

3. `monitoring-edge-basic` (Private recommended)
   - Description: "Edge monitoring agent for Docker hosts"
   - Initialize: No

4. `monitoring-edge-postgres` (Private recommended)
   - Description: "Edge monitoring agent with PostgreSQL/TimescaleDB monitoring"
   - Initialize: No

### 1.2 Note Repository URLs

After creating, note the Git URLs:
```
https://github.com/YOUR-USERNAME/monitoring-stack.git
https://github.com/YOUR-USERNAME/monitoring-grafana.git
https://github.com/YOUR-USERNAME/monitoring-edge-basic.git
https://github.com/YOUR-USERNAME/monitoring-edge-postgres.git
```

## Step 2: Push Stack Code to GitHub

### 2.1 Push Central Stack

```bash
cd /path/to/monitoring-stack/grafana

# Initialize git
git init
git branch -M main

# Add files
git add .
git commit -m "Initial commit: Central monitoring stack

- Caddy for SSL and reverse proxy
- Prometheus with remote write receiver (2y retention)
- Loki for log aggregation (3 months retention)
- Grafana with auto-provisioned datasources
- Promtail for grafana server logs"

# Add remote and push
git remote add origin https://github.com/YOUR-USERNAME/monitoring-grafana.git
git push -u origin main
```

### 2.2 Push Edge-Basic Stack

```bash
cd /path/to/monitoring-stack/edge-basic

# Initialize git
git init
git branch -M main

# Add files
git add .
git commit -m "Initial commit: Edge monitoring stack (basic)

- Prometheus agent with remote write (0 retention)
- Promtail for log collection
- cAdvisor for container metrics
- Node Exporter for host metrics"

# Add remote and push
git remote add origin https://github.com/YOUR-USERNAME/monitoring-edge-basic.git
git push -u origin main
```

### 2.3 Push Edge-Postgres Stack

```bash
cd /path/to/monitoring-stack/edge-postgres

# Initialize git
git init
git branch -M main

# Add files
git add .
git commit -m "Initial commit: Edge monitoring stack (PostgreSQL)

- All features from edge-basic
- PostgreSQL Exporter with custom TimescaleDB queries
- Connection to dispatch-network for PostgreSQL access"

# Add remote and push
git remote add origin https://github.com/YOUR-USERNAME/monitoring-edge-postgres.git
git push -u origin main
```

### 2.4 Push Umbrella Repository

```bash
cd /path/to/monitoring-stack

# Remove stack directories (they'll be added as submodules)
rm -rf grafana edge-basic edge-postgres

# Initialize git (if not already)
git init
git branch -M main

# Add files
git add .
git commit -m "Initial commit: Monitoring stack umbrella repository

- Architecture documentation
- Setup and deployment guides
- VPC configuration instructions
- Troubleshooting guide"

# Add remote and push
git remote add origin https://github.com/YOUR-USERNAME/monitoring-stack.git
git push -u origin main
```

## Step 3: Add Submodules to Umbrella Repo

Now add the three stack repositories as submodules:

```bash
cd /path/to/monitoring-stack

# Add grafana stack as submodule
git submodule add https://github.com/YOUR-USERNAME/monitoring-grafana.git central

# Add edge-basic stack as submodule
git submodule add https://github.com/YOUR-USERNAME/monitoring-edge-basic.git edge-basic

# Add edge-postgres stack as submodule
git submodule add https://github.com/YOUR-USERNAME/monitoring-edge-postgres.git edge-postgres

# Commit submodule additions
git add .gitmodules grafana edge-basic edge-postgres
git commit -m "Add monitoring stacks as submodules"
git push origin main
```

## Step 4: Verify Setup

### 4.1 Check Submodules

```bash
# List submodules
git submodule status

# Should show:
#  <commit-hash> grafana (heads/main)
#  <commit-hash> edge-basic (heads/main)
#  <commit-hash> edge-postgres (heads/main)
```

### 4.2 Check .gitmodules File

```bash
cat .gitmodules
```

Should contain:
```
[submodule "central"]
    path = central
    url = https://github.com/YOUR-USERNAME/monitoring-grafana.git
[submodule "edge-basic"]
    path = edge-basic
    url = https://github.com/YOUR-USERNAME/monitoring-edge-basic.git
[submodule "edge-postgres"]
    path = edge-postgres
    url = https://github.com/YOUR-USERNAME/monitoring-edge-postgres.git
```

## Step 5: Clone and Work with Repository

### 5.1 Clone with Submodules (Fresh Clone)

```bash
# Clone umbrella repo with all submodules
git clone --recurse-submodules https://github.com/YOUR-USERNAME/monitoring-stack.git

cd monitoring-stack

# Verify submodules are populated
ls grafana edge-basic edge-postgres
```

### 5.2 Update Existing Clone (If Already Cloned)

```bash
cd monitoring-stack

# Initialize and update submodules
git submodule init
git submodule update --recursive

# Or in one command:
git submodule update --init --recursive
```

## Step 6: Working with Submodules

### 6.1 Make Changes to a Stack

```bash
cd monitoring-stack/grafana

# Make changes
vim docker-compose.yml

# Commit in submodule
git add docker-compose.yml
git commit -m "Update Prometheus retention to 1 year"
git push origin main

# Update umbrella repo to track new commit
cd ..
git add central
git commit -m "Update grafana stack to latest version"
git push origin main
```

### 6.2 Update All Submodules to Latest

```bash
cd monitoring-stack

# Update all submodules to latest commits
git submodule update --remote --merge

# Commit the updates
git add grafana edge-basic edge-postgres
git commit -m "Update all stacks to latest versions"
git push origin main
```

### 6.3 Update Specific Submodule

```bash
cd monitoring-stack

# Update only grafana stack
git submodule update --remote --merge central

# Commit
git add central
git commit -m "Update grafana stack"
git push origin main
```

## Step 7: Portainer GitOps Setup

Now that repositories are on GitHub, configure Portainer:

### 7.1 Central Stack in Portainer

1. Portainer → Stacks → Add stack
2. Select "Git Repository"
3. Configure:
   - **Repository URL**: `https://github.com/YOUR-USERNAME/monitoring-grafana`
   - **Branch**: `main`
   - **Compose path**: `docker-compose.yml`
   - **Authentication**: If private repo, add credentials
4. Add environment variables
5. Enable **Polling** for GitOps (e.g., 5 minutes)
6. Deploy

### 7.2 Edge Stacks in Portainer

Repeat for each edge host:

**For basic edge:**
- Repository: `https://github.com/YOUR-USERNAME/monitoring-edge-basic`

**For PostgreSQL edge:**
- Repository: `https://github.com/YOUR-USERNAME/monitoring-edge-postgres`

## Troubleshooting

### Submodule Not Initialized

```bash
# If submodule directory is empty
git submodule update --init --recursive
```

### Detached HEAD in Submodule

```bash
cd submodule-directory
git checkout main
git pull origin main
cd ..
git add submodule-directory
git commit -m "Update submodule to track main branch"
```

### Remove Submodule (If Needed)

```bash
# Remove submodule entry from .gitmodules
git config -f .gitmodules --remove-section submodule.central

# Remove submodule entry from .git/config
git config -f .git/config --remove-section submodule.central

# Remove submodule directory
git rm --cached central
rm -rf central

# Commit removal
git commit -m "Remove grafana submodule"
```

### Submodule Push Permission Denied

Ensure you have write access to the submodule repository. If private, you may need to:
1. Use SSH URLs instead of HTTPS
2. Configure Git credentials
3. Use Personal Access Token

## Alternative: SSH URLs

If you prefer SSH over HTTPS:

### Convert to SSH URLs

```bash
# Edit .gitmodules
vim .gitmodules

# Change URLs from:
# url = https://github.com/USER/repo.git
# To:
# url = git@github.com:USER/repo.git

# Sync configuration
git submodule sync

# Update submodules
git submodule update --init --recursive
```

## Best Practices

1. **Always commit in submodule first**, then update umbrella repo
2. **Use GitOps carefully**: Pushes trigger auto-deploys in Portainer
3. **Test locally** before pushing to production branches
4. **Use branches** for testing (e.g., `dev`, `staging`)
5. **Tag releases**: `git tag v1.0.0 && git push --tags`
6. **Document changes**: Write clear commit messages
7. **Regular updates**: Keep submodules up to date

## GitHub Actions (Optional)

You can add CI/CD workflows to validate changes:

### Example: .github/workflows/validate.yml

```yaml
name: Validate Stack

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Validate docker-compose
        run: |
          docker-compose config

      - name: Check for secrets
        run: |
          ! grep -r "password.*=" . --include="*.yml" || exit 1
```

Add this to each stack repository for automated validation.

## Further Reading

- [Git Submodules Official Docs](https://git-scm.com/book/en/v2/Git-Tools-Submodules)
- [Portainer GitOps](https://docs.portainer.io/user/docker/stacks/add#git-repository)
- [GitHub SSH Setup](https://docs.github.com/en/authentication/connecting-to-github-with-ssh)
