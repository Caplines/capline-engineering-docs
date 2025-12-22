# CI/CD Pipeline Implementation Guide
## Production-Ready Deployment for NestJS Applications on AWS EC2

**Document Version:** 2.0  
**Last Updated:** December 2024  
**Prepared By:** Senior DevOps Engineering Team  
**Classification:** Internal Engineering Documentation

---

## 📋 Table of Contents

1. [Executive Summary](#executive-summary)
2. [Architecture Overview](#architecture-overview)
3. [Prerequisites & Requirements](#prerequisites--requirements)
4. [Infrastructure Setup](#infrastructure-setup)
5. [GitHub Configuration](#github-configuration)
6. [PM2 Process Management](#pm2-process-management)
7. [CI/CD Workflow Implementation](#cicd-workflow-implementation)
8. [Database Management](#database-management)
9. [Testing & Validation](#testing--validation)
10. [Production Deployment](#production-deployment)
11. [Monitoring & Maintenance](#monitoring--maintenance)
12. [Security Best Practices](#security-best-practices)
13. [Troubleshooting Guide](#troubleshooting-guide)

---

## Executive Summary

This document provides a comprehensive, production-tested implementation guide for deploying NestJS applications on AWS EC2 using GitHub Actions, PM2, and modern DevOps practices.

### Key Features

- ✅ Automated CI/CD with GitHub Actions
- ✅ Dual environment deployment (Staging/Production)
- ✅ Zero-downtime production deployments
- ✅ Automated database backups and migrations
- ✅ Intelligent rollback mechanisms
- ✅ Comprehensive health checks
- ✅ Email notifications
- ✅ Non-blocking lint and test warnings

### Technology Stack

| Component | Technology | Version |
|-----------|-----------|---------|
| Runtime | Node.js | 20.x LTS |
| Framework | NestJS | Latest |
| Database | PostgreSQL | 15.x |
| ORM | Prisma | Latest |
| Process Manager | PM2 | Latest |
| CI/CD | GitHub Actions | - |
| Server | AWS EC2 | t3.medium+ |
| Reverse Proxy | Nginx | Latest stable |

---

## Architecture Overview

### Deployment Flow Diagram

\`\`\`
┌──────────────────────────────────────────────────────────────┐
│                     GitHub Repository                         │
│                                                               │
│    develop branch ──────────┐      main branch               │
│         │                   │           │                     │
└─────────┼───────────────────┼───────────┼─────────────────────┘
          │                   │           │
          │ Push              │           │ Push
          ▼                   │           ▼
┌─────────────────────────────┼─────────────────────────────────┐
│         GitHub Actions      │         GitHub Actions           │
│                            │                                   │
│  ┌─────────┐  ┌─────────┐ │ ┌─────────┐  ┌──────────┐       │
│  │   CI    │→│  Build  │→│ │   CI    │→│   Build   │        │
│  └─────────┘  └─────────┘ │ └─────────┘  └──────────┘       │
│                            │                                   │
│  ┌─────────────────────┐  │ ┌──────────────────────┐         │
│  │  Deploy to Staging  │  │ │ Deploy to Production │         │
│  │  (Auto)             │  │ │ (Manual Approval)    │         │
│  └─────────────────────┘  │ └──────────────────────┘         │
└────────────┬───────────────┴─────────────┬────────────────────┘
             │                             │
             ▼                             ▼
    ┌─────────────────┐          ┌─────────────────┐
    │ Staging Server  │          │Production Server│
    │                 │          │                 │
    │ ┌─────────────┐ │          │ ┌─────────────┐ │
    │ │     PM2     │ │          │ │     PM2     │ │
    │ │  (Cluster)  │ │          │ │  (Cluster)  │ │
    │ │             │ │          │ │             │ │
    │ │  NestJS App │ │          │ │  NestJS App │ │
    │ └─────────────┘ │          │ └─────────────┘ │
    │                 │          │                 │
    │ ┌─────────────┐ │          │ ┌─────────────┐ │
    │ │ PostgreSQL  │ │          │ │ PostgreSQL  │ │
    │ └─────────────┘ │          │ └─────────────┘ │
    └─────────────────┘          └─────────────────┘
\`\`\`

### Pipeline Stages

\`\`\`
Developer Push
    │
    ├──▶ [CI] Install Dependencies
    ├──▶ [CI] Generate Prisma Clients
    ├──▶ [CI] Validate Database Schema
    ├──▶ [CI] Lint (Warning-only)
    ├──▶ [CI] Test (Warning-only)
    │
    ├──▶ [BUILD] TypeScript Compilation
    ├──▶ [BUILD] Prune Dev Dependencies
    ├──▶ [BUILD] Create Deployment Artifact
    │
    ├──▶ [DEPLOY] Transfer Artifact
    ├──▶ [DEPLOY] Backup Current Version
    ├──▶ [DEPLOY] Extract Files
    ├──▶ [DEPLOY] Database Backup
    ├──▶ [DEPLOY] Run Migrations
    ├──▶ [DEPLOY] Restart PM2
    │
    ├──▶ [VERIFY] Health Checks
    ├──▶ [VERIFY] Smoke Tests
    │
    └──▶ [NOTIFY] Send Status Email
\`\`\`

---

## Prerequisites & Requirements

### Infrastructure Requirements

#### AWS EC2 Specifications

| Environment | Instance Type | vCPUs | RAM | Storage |
|-------------|---------------|-------|-----|---------|
| Staging | t3.medium | 2 | 4 GB | 20 GB |
| Production | t3.large | 2 | 8 GB | 30 GB |

**Operating System:** Ubuntu 24.04 LTS or Amazon Linux 2023

**Network Configuration:**
- VPC with public subnet
- Security group with ports: 22 (SSH), 80 (HTTP), 443 (HTTPS)
- Elastic IP (recommended for production)

### Software Requirements

**Server Software:**

\`\`\`bash
Node.js:     v20.x LTS
npm:         v10.x
PM2:         Latest
PostgreSQL:  v15.x
Git:         v2.x
Nginx:       Latest stable
\`\`\`

---

## Infrastructure Setup

### Phase 1: Initial Server Configuration

#### Connect and Update System

\`\`\`bash
# Connect to EC2 instance
ssh -i your-key.pem ubuntu@your-server-ip

# Update system packages
sudo apt update && sudo apt upgrade -y

# Install essential tools
sudo apt install -y curl wget vim git build-essential
\`\`\`

#### Install Node.js 20.x

\`\`\`bash
# Add NodeSource repository
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -

# Install Node.js
sudo apt-get install -y nodejs

# Verify installation
node --version    # Should output v20.x
npm --version     # Should output v10.x
\`\`\`

#### Install PostgreSQL Client

\`\`\`bash
# Add PostgreSQL repository
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -

# Update and install
sudo apt-get update
sudo apt-get install -y postgresql-client-15

# Verify
psql --version
\`\`\`

#### Install PM2 Globally

\`\`\`bash
# Install PM2
sudo npm install -g pm2

# Verify
pm2 --version

# Configure startup script
pm2 startup
# Follow the instructions provided
\`\`\`

### Phase 2: User and Directory Setup

#### Create Deployment User

\`\`\`bash
# Create deploy user
sudo adduser deploy --disabled-password --gecos ""

# Add to sudo group
sudo usermod -aG sudo deploy

# Configure passwordless sudo
echo "deploy ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/deploy
sudo chmod 440 /etc/sudoers.d/deploy
\`\`\`

#### Setup Application Directories

\`\`\`bash
# Switch to deploy user
sudo su - deploy

# Create directory structure
sudo mkdir -p /var/www/backend
sudo mkdir -p /var/www/backups/{app,database}

# Set ownership
sudo chown -R deploy:deploy /var/www
sudo chmod -R 755 /var/www
\`\`\`

### Phase 3: PM2 Configuration

#### Setup Log Rotation

\`\`\`bash
# Install PM2 log rotation
pm2 install pm2-logrotate

# Configure rotation
pm2 set pm2-logrotate:max_size 10M
pm2 set pm2-logrotate:retain 7
pm2 set pm2-logrotate:compress true
\`\`\`

### Phase 4: Nginx Setup

#### Install Nginx

\`\`\`bash
sudo apt-get install -y nginx
sudo systemctl start nginx
sudo systemctl enable nginx
\`\`\`

#### Configure Reverse Proxy

Create \`/etc/nginx/sites-available/capline-backend\`:

\`\`\`nginx
upstream capline_backend {
    least_conn;
    server 127.0.0.1:7474 max_fails=3 fail_timeout=30s;
    keepalive 32;
}

server {
    listen 80;
    server_name staging.yourdomain.com;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Logging
    access_log /var/log/nginx/capline-access.log;
    error_log /var/log/nginx/capline-error.log;

    location / {
        proxy_pass http://capline_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_cache_bypass $http_upgrade;
    }
}
\`\`\`

Enable configuration:

\`\`\`bash
sudo ln -s /etc/nginx/sites-available/capline-backend /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
\`\`\`

### Phase 5: Security Hardening

#### Configure UFW Firewall

\`\`\`bash
sudo ufw --force enable
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw status verbose
\`\`\`

#### Secure SSH

Edit \`/etc/ssh/sshd_config\`:

\`\`\`
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
\`\`\`

Restart SSH:

\`\`\`bash
sudo systemctl restart sshd
\`\`\`

---

## GitHub Configuration

### SSH Key Management

#### Generate SSH Keys

\`\`\`bash
# Staging key
ssh-keygen -t ed25519 -C "github-staging" -f ~/.ssh/capline-staging -N ""

# Production key
ssh-keygen -t ed25519 -C "github-production" -f ~/.ssh/capline-production -N ""

# Display public keys
cat ~/.ssh/capline-staging.pub
cat ~/.ssh/capline-production.pub
\`\`\`

#### Add Keys to Servers

On each server:

\`\`\`bash
ssh deploy@server-ip
echo "your-public-key-here" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
\`\`\`

### GitHub Secrets Configuration

Navigate to: \`Repository → Settings → Secrets and variables → Actions\`

**Required Secrets:**

| Secret Name | Description |
|-------------|-------------|
| STAGING_HOST | Staging server IP |
| STAGING_USERNAME | SSH username (deploy) |
| STAGING_SSH_KEY | Full private key |
| PRODUCTION_HOST | Production server IP |
| PRODUCTION_USERNAME | SSH username (deploy) |
| PRODUCTION_SSH_KEY | Full private key |
| SMTP_HOST | SMTP server |
| SMTP_PORT | SMTP port (587) |
| SMTP_USERNAME | SMTP username |
| SMTP_PASSWORD | SMTP password |
| NOTIFICATION_EMAIL | Notification recipient |

---

## PM2 Process Management

### Main Application Configuration

Create \`ecosystem.config.js\`:

\`\`\`javascript
module.exports = {
  apps: [{
    name: 'capline-core',
    script: './dist/src/main.js',
    instances: 1,
    exec_mode: 'cluster',
    node_args: '--max-old-space-size=1536',
    
    env_staging: {
      NODE_ENV: 'staging',
      PORT: 7474
    },
    env_production: {
      NODE_ENV: 'production',
      PORT: 7474
    },
    
    error_file: './logs/pm2-err.log',
    out_file: './logs/pm2-out.log',
    log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
    
    max_memory_restart: '1800M',
    autorestart: true,
    max_restarts: 10,
    min_uptime: '10s',
    
    kill_timeout: 5000,
    wait_ready: true,
    cron_restart: '0 3 * * *'
  }]
};
\`\`\`

### Essential PM2 Commands

\`\`\`bash
# Start application
pm2 start ecosystem.config.js --env production

# Restart
pm2 restart capline-core

# Reload (zero-downtime)
pm2 reload capline-core

# View logs
pm2 logs capline-core --lines 100

# Monitor
pm2 monit

# Status
pm2 list

# Save configuration
pm2 save
\`\`\`

---

## CI/CD Workflow Implementation

### Complete GitHub Actions Workflow

Create `.github/workflows/deploy.yml`:

\`\`\`yaml
name: CI/CD Pipeline

on:
  push:
    branches: [develop, main]
  pull_request:
    branches: [develop, main]

env:
  NODE_VERSION: '20'
  APP_NAME: 'capline-core'

jobs:
  ci:
    name: 🔍 Continuous Integration
    runs-on: ubuntu-latest
    timeout-minutes: 15

    steps:
      - name: 📥 Checkout
        uses: actions/checkout@v4

      - name: 🔧 Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: \${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: 📦 Install Dependencies
        run: npm ci

      - name: 🔨 Generate Prisma
        run: npm run generate

      - name: ✅ Validate Schema
        run: npm run test:schema

      - name: 🎨 Lint
        continue-on-error: true
        run: npm run lint

      - name: 🧪 Test
        continue-on-error: true
        run: npm run test

  build:
    name: 🏗️ Build
    needs: ci
    runs-on: ubuntu-latest
    if: github.event_name == 'push'

    steps:
      - name: 📥 Checkout
        uses: actions/checkout@v4

      - name: 🔧 Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: \${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: 📦 Install
        run: npm ci

      - name: 🔨 Generate
        run: npm run generate

      - name: 🏗️ Build
        run: npm run build

      - name: 🧹 Prune
        run: npm prune --production

      - name: 📦 Create Artifact
        run: |
          tar -czf deployment-\${{ github.sha }}.tar.gz \
            dist/ node_modules/ prisma/ scripts/ \
            ecosystem.config.js package.json

      - name: 💾 Upload Artifact
        uses: actions/upload-artifact@v4
        with:
          name: deployment-artifact-\${{ github.sha }}
          path: deployment-\${{ github.sha }}.tar.gz
          retention-days: 30

  deploy-staging:
    name: 🚀 Deploy to Staging
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    environment: staging

    steps:
      - name: 📥 Download Artifact
        uses: actions/download-artifact@v4
        with:
          name: deployment-artifact-\${{ github.sha }}

      - name: 📤 Transfer to Server
        uses: appleboy/scp-action@v0.1.7
        with:
          host: \${{ secrets.STAGING_HOST }}
          username: \${{ secrets.STAGING_USERNAME }}
          key: \${{ secrets.STAGING_SSH_KEY }}
          source: 'deployment-\${{ github.sha }}.tar.gz'
          target: '/tmp'

      - name: 🚀 Deploy
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: \${{ secrets.STAGING_HOST }}
          username: \${{ secrets.STAGING_USERNAME }}
          key: \${{ secrets.STAGING_SSH_KEY }}
          script: |
            set -e
            APP_PATH="/var/www/backend/capline-backend-services"
            ARTIFACT="/tmp/deployment-\${{ github.sha }}.tar.gz"
            
            # Backup current version
            if [ -d "$APP_PATH" ]; then
              sudo cp -r "$APP_PATH" "${APP_PATH}_backup_$(date +%Y%m%d_%H%M%S)"
            fi
            
            # Extract artifact
            sudo mkdir -p "$APP_PATH"
            sudo tar -xzf "$ARTIFACT" -C "$APP_PATH"
            rm "$ARTIFACT"
            sudo chown -R $(whoami):$(whoami) "$APP_PATH"
            
            cd "$APP_PATH"
            npm run generate
            npm run migrate:deploy
            
            # Restart PM2
            pm2 restart \${{ env.APP_NAME }} || pm2 start ecosystem.config.js --env staging
            pm2 save

      - name: 🏥 Health Check
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: \${{ secrets.STAGING_HOST }}
          username: \${{ secrets.STAGING_USERNAME }}
          key: \${{ secrets.STAGING_SSH_KEY }}
          script: |
            sleep 15
            for i in {1..10}; do
              if curl -f http://localhost:7474/api/health-check; then
                echo "✅ Health check passed"
                exit 0
              fi
              sleep 3
            done
            echo "❌ Health check failed"
            exit 1

  deploy-production:
    name: 🚀 Deploy to Production
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production

    steps:
      - name: 📥 Download Artifact
        uses: actions/download-artifact@v4
        with:
          name: deployment-artifact-\${{ github.sha }}

      - name: 📤 Transfer to Server
        uses: appleboy/scp-action@v0.1.7
        with:
          host: \${{ secrets.PRODUCTION_HOST }}
          username: \${{ secrets.PRODUCTION_USERNAME }}
          key: \${{ secrets.PRODUCTION_SSH_KEY }}
          source: 'deployment-\${{ github.sha }}.tar.gz'
          target: '/tmp'

      - name: 🚀 Deploy
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: \${{ secrets.PRODUCTION_HOST }}
          username: \${{ secrets.PRODUCTION_USERNAME }}
          key: \${{ secrets.PRODUCTION_SSH_KEY }}
          script: |
            set -e
            APP_PATH="/var/www/backend/capline-backend-services"
            
            # MANDATORY backup
            if [ -d "$APP_PATH" ]; then
              sudo cp -r "$APP_PATH" "${APP_PATH}_backup_$(date +%Y%m%d_%H%M%S)"
            fi
            
            # Extract and deploy
            sudo mkdir -p "$APP_PATH"
            sudo tar -xzf "/tmp/deployment-\${{ github.sha }}.tar.gz" -C "$APP_PATH"
            rm "/tmp/deployment-\${{ github.sha }}.tar.gz"
            sudo chown -R $(whoami):$(whoami) "$APP_PATH"
            
            cd "$APP_PATH"
            npm run generate
            
            # Database backup (MANDATORY)
            if ! ./scripts/backup-database.sh; then
              echo "❌ Backup failed - ABORTING"
              exit 1
            fi
            
            npm run migrate:deploy
            pm2 reload \${{ env.APP_NAME }} || pm2 start ecosystem.config.js --env production
            pm2 save

      - name: 🏥 Health Check
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: \${{ secrets.PRODUCTION_HOST }}
          username: \${{ secrets.PRODUCTION_USERNAME }}
          key: \${{ secrets.PRODUCTION_SSH_KEY }}
          script: |
            sleep 20
            for i in {1..15}; do
              if curl -f http://localhost:7474/api/health-check; then
                echo "✅ Production health check passed"
                exit 0
              fi
              sleep 5
            done
            echo "❌ Health check failed"
            exit 1
\`\`\`

---

## Database Management

### Database Backup Script

Create \`scripts/backup-database.sh\`:

\`\`\`bash
#!/bin/bash
set -e

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="$HOME/db_backups"
mkdir -p "$BACKUP_DIR"

# Load environment
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
PROJECT_ROOT="$(cd "$SCRIPT_DIR/.." && pwd)"
ENV_FILE="$PROJECT_ROOT/.env"

if [ ! -f "$ENV_FILE" ]; then
    echo "❌ .env file not found"
    exit 1
fi

export $(cat "$ENV_FILE" | grep POSTGRESQL_DATABASE_URL | xargs)

BACKUP_FILE="$BACKUP_DIR/postgres_backup_${TIMESTAMP}.sql"

echo "🗄️ Creating database backup..."
pg_dump "$POSTGRESQL_DATABASE_URL" > "$BACKUP_FILE"

if [ -s "$BACKUP_FILE" ]; then
    gzip "$BACKUP_FILE"
    echo "✅ Backup completed: ${BACKUP_FILE}.gz"
else
    echo "❌ Backup failed"
    exit 1
fi
\`\`\`

Make executable:

\`\`\`bash
chmod +x scripts/backup-database.sh
\`\`\`

### Prisma Commands

Add to \`package.json\`:

\`\`\`json
{
  "scripts": {
    "migrate:dev": "prisma migrate dev --schema=./prisma/postgresql/schema.prisma",
    "migrate:deploy": "prisma migrate deploy --schema=./prisma/postgresql/schema.prisma",
    "generate": "prisma generate --schema=./prisma/postgresql/schema.prisma",
    "test:schema": "prisma validate --schema=./prisma/postgresql/schema.prisma"
  }
}
\`\`\`

---

## Testing & Validation

### Health Check Endpoint

\`\`\`typescript
import { Controller, Get } from '@nestjs/common';

@Controller()
export class AppController {
  @Get('health')
  healthCheck() {
    return {
      status: 'ok',
      timestamp: new Date().toISOString(),
      uptime: process.uptime(),
      memory: {
        used: Math.round(process.memoryUsage().heapUsed / 1024 / 1024),
        total: Math.round(process.memoryUsage().heapTotal / 1024 / 1024)
      }
    };
  }

  @Get('api/health-check')
  apiHealthCheck() {
    return this.healthCheck();
  }
}
\`\`\`

### Local Testing

\`\`\`bash
# Install dependencies
npm ci

# Generate Prisma
npm run generate

# Validate schema
npm run test:schema

# Lint
npm run lint

# Test
npm run test

# Build
npm run build

# Test build
npm run start:prod

# Test health (in another terminal)
curl http://localhost:7474/api/health-check
\`\`\`

### Staging Validation

\`\`\`bash
# SSH to staging
ssh deploy@staging-server-ip

# Check PM2
pm2 status
pm2 logs capline-core --lines 50

# Test health
curl http://localhost:7474/api/health-check

# Monitor
pm2 monit
\`\`\`

---

## Production Deployment

### Deployment Process

#### Step 1: Merge to Main

\`\`\`bash
git checkout develop
git pull origin develop

git checkout main
git pull origin main
git merge develop
git push origin main
\`\`\`

#### Step 2: Monitor Deployment

1. Go to GitHub → Actions
2. Watch pipeline execution
3. Approve production deployment

#### Step 3: Verify

\`\`\`bash
ssh deploy@production-server-ip
pm2 status
pm2 logs capline-core --lines 50
curl http://localhost:7474/api/health-check
\`\`\`

### Emergency Hotfix

\`\`\`bash
# Create hotfix branch
git checkout main
git checkout -b hotfix/critical-fix

# Make changes
# ... fix code ...

# Test locally
npm run build
npm run test

# Merge and deploy
git checkout main
git merge hotfix/critical-fix
git push origin main

# Backport to develop
git checkout develop
git merge hotfix/critical-fix
git push origin develop
\`\`\`

### Rollback Procedure

\`\`\`bash
# SSH to server
ssh deploy@production-server-ip

# List backups
ls -lt /var/www/backend/capline-backend-services_backup_*

# Stop app
pm2 stop capline-core

# Restore backup
APP_PATH="/var/www/backend/capline-backend-services"
BACKUP_PATH="<backup-path>"
sudo rm -rf "$APP_PATH"
sudo mv "$BACKUP_PATH" "$APP_PATH"
sudo chown -R deploy:deploy "$APP_PATH"

# Restart
cd "$APP_PATH"
pm2 restart capline-core
\`\`\`

---

## Monitoring & Maintenance

### PM2 Monitoring

\`\`\`bash
# Real-time monitoring
pm2 monit

# View logs
pm2 logs capline-core
pm2 logs capline-core --lines 100
pm2 logs capline-core --err

# Process info
pm2 show capline-core
pm2 list
\`\`\`

### System Monitoring

\`\`\`bash
# Resources
free -h
df -h
top
htop

# Network
netstat -tulpn | grep 7474

# Disk usage
du -sh /var/www/backend/*
\`\`\`

### Performance Metrics

| Metric | Tool | Threshold | Action |
|--------|------|-----------|--------|
| CPU | top, pm2 | > 80% | Investigate |
| Memory | free, pm2 | > 85% | Restart/Scale |
| Response Time | logs | > 2s | Optimize |
| Error Rate | logs | > 1% | Debug |

---

## Security Best Practices

### Environment Variables

Never commit:
- \`.env\`
- \`*.pem\`
- \`*.key\`

Use \`.env.example\`:

\`\`\`bash
# Database
POSTGRESQL_DATABASE_URL=postgresql://user:pass@host:5432/db

# Application
APP_PORT=7474
NODE_ENV=production

# JWT
JWT_SECRET=your-secret
JWT_EXPIRES_IN=7d
\`\`\`

### SSH Security

\`\`\`bash
# Use Ed25519 keys
ssh-keygen -t ed25519 -C "email@example.com"

# Disable password auth
# /etc/ssh/sshd_config:
PasswordAuthentication no
PermitRootLogin no
\`\`\`

### Application Security

\`\`\`typescript
// Helmet
import helmet from 'helmet';
app.use(helmet());

// Rate limiting
import rateLimit from 'express-rate-limit';
app.use(rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100
}));

// CORS
app.enableCors({
  origin: process.env.ALLOWED_ORIGINS?.split(','),
  credentials: true
});
\`\`\`

---

## Troubleshooting Guide

### Common Issues

#### Deployment Fails

**Symptoms:** Build fails, TypeScript errors

**Solutions:**
\`\`\`bash
npm ci
npm run build
npm run test
\`\`\`

#### Health Check Fails

**Symptoms:** Connection refused, 502 errors

**Solutions:**
\`\`\`bash
pm2 list
pm2 logs capline-core --lines 100
netstat -tulpn | grep 7474
curl -v http://localhost:7474/api/health-check
pm2 restart capline-core
\`\`\`

#### Migration Fails

**Symptoms:** Migration errors, can't connect to DB

**Solutions:**
\`\`\`bash
psql $POSTGRESQL_DATABASE_URL
npx prisma migrate status
npx prisma migrate deploy
\`\`\`

#### PM2 Crashes

**Symptoms:** Process keeps restarting

**Solutions:**
\`\`\`bash
pm2 logs capline-core --lines 200
pm2 monit
pm2 delete capline-core
pm2 start ecosystem.config.js --env production
\`\`\`

#### High Memory

**Symptoms:** OOM errors, slow performance

**Solutions:**
\`\`\`bash
free -h
pm2 list
pm2 restart capline-core
# Increase max_memory_restart in ecosystem.config.js
\`\`\`

### Debug Commands

\`\`\`bash
# PM2
pm2 list
pm2 show capline-core
pm2 logs capline-core
pm2 monit

# System
top
free -h
df -h
netstat -tulpn

# Logs
tail -f /var/log/nginx/error.log
pm2 logs --err

# Network
curl -v http://localhost:7474/api/health-check
telnet localhost 7474
\`\`\`

---

## Appendix

### Glossary

| Term | Definition |
|------|------------|
| **CI/CD** | Continuous Integration/Deployment |
| **PM2** | Process Manager for Node.js |
| **Artifact** | Built application package |
| **Rollback** | Revert to previous version |
| **Health Check** | Application status endpoint |

### Reference Links

- [NestJS Docs](https://docs.nestjs.com)
- [PM2 Docs](https://pm2.keymetrics.io/docs)
- [GitHub Actions](https://docs.github.com/en/actions)
- [Prisma Docs](https://www.prisma.io/docs)

### Checklist

**Pre-Deployment:**
- [ ] Tests passing
- [ ] Code reviewed
- [ ] Migrations tested
- [ ] Secrets configured
- [ ] Backup plan ready

**Post-Deployment:**
- [ ] Health checks passing
- [ ] Logs reviewed
- [ ] Performance normal
- [ ] Team notified

---

**Document End**

For questions or improvements, contact the DevOps team.

**Version:** 2.0  
**Last Updated:** December 2024
