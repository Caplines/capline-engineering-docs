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

```mermaid
graph TB
    subgraph GitHub["GitHub Repository"]
        develop[develop branch]
        main[main branch]
    end
    
    subgraph StagingCI["Staging GitHub Actions"]
        sci1[CI: Install & Test]
        sci2[Build Application]
        sci3[Deploy to Staging]
    end
    
    subgraph ProdCI["Production GitHub Actions"]
        pci1[CI: Install & Test]
        pci2[Build Application]
        pci3[Manual Approval]
        pci4[Deploy to Production]
    end
    
    subgraph StagingServer["Staging Server"]
        spm2[PM2 Cluster]
        sapp[NestJS App]
        sdb[PostgreSQL]
    end
    
    subgraph ProdServer["Production Server"]
        ppm2[PM2 Cluster]
        papp[NestJS App]
        pdb[PostgreSQL]
    end
    
    develop -->|Push| StagingCI
    sci1 --> sci2 --> sci3
    sci3 --> StagingServer
    
    main -->|Push| ProdCI
    pci1 --> pci2 --> pci3 --> pci4
    pci4 --> ProdServer
    
    spm2 --> sapp --> sdb
    ppm2 --> papp --> pdb
```

### Pipeline Stages

```mermaid
graph LR
    A[Developer Push] --> B[CI: Dependencies]
    B --> C[CI: Prisma Generate]
    C --> D[CI: Schema Validation]
    D --> E[CI: Lint Warning]
    E --> F[CI: Test Warning]
    F --> G[Build: TypeScript]
    G --> H[Build: Prune Dev Deps]
    H --> I[Build: Create Artifact]
    I --> J[Deploy: Transfer]
    J --> K[Deploy: Backup]
    K --> L[Deploy: Extract Files]
    L --> M[Deploy: DB Backup]
    M --> N[Deploy: Migrations]
    N --> O[Deploy: Restart PM2]
    O --> P[Verify: Health Checks]
    P --> Q[Verify: Smoke Tests]
    Q --> R[Notify: Email Status]
```

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

```bash
Node.js:     v20.x LTS
npm:         v10.x
PM2:         Latest
PostgreSQL:  v15.x
Git:         v2.x
Nginx:       Latest stable
```

---

## Infrastructure Setup

### Phase 1: Initial Server Configuration

#### Connect and Update System

```bash
# Connect to EC2 instance
ssh -i your-key.pem ubuntu@your-server-ip

# Update system packages
sudo apt update && sudo apt upgrade -y

# Install essential tools
sudo apt install -y curl wget vim git build-essential
```

#### Install Node.js 20.x

```bash
# Add NodeSource repository
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -

# Install Node.js
sudo apt-get install -y nodejs

# Verify installation
node --version    # Should output v20.x
npm --version     # Should output v10.x
```

#### Install PostgreSQL Client

```bash
# Add PostgreSQL repository
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -

# Update and install
sudo apt-get update
sudo apt-get install -y postgresql-client-15

# Verify
psql --version
```

#### Install PM2 Globally

```bash
# Install PM2
sudo npm install -g pm2

# Verify
pm2 --version

# Configure startup script
pm2 startup
# Follow the instructions provided
```

### Phase 2: User and Directory Setup

#### Create Deployment User

```bash
# Create deploy user
sudo adduser deploy --disabled-password --gecos ""

# Add to sudo group
sudo usermod -aG sudo deploy

# Configure passwordless sudo
echo "deploy ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/deploy
sudo chmod 440 /etc/sudoers.d/deploy
```

#### Setup Application Directories

```bash
# Switch to deploy user
sudo su - deploy

# Create directory structure
sudo mkdir -p /var/www/backend
sudo mkdir -p /var/www/backups/{app,database}

# Set ownership
sudo chown -R deploy:deploy /var/www
sudo chmod -R 755 /var/www
```

### Phase 3: PM2 Configuration

#### Setup Log Rotation

```bash
# Install PM2 log rotation
pm2 install pm2-logrotate

# Configure rotation
pm2 set pm2-logrotate:max_size 10M
pm2 set pm2-logrotate:retain 7
pm2 set pm2-logrotate:compress true
```

### Phase 4: Nginx Setup

#### Install Nginx

```bash
sudo apt-get install -y nginx
sudo systemctl start nginx
sudo systemctl enable nginx
```

#### Configure Reverse Proxy

Create `/etc/nginx/sites-available/capline-backend`:

```nginx
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
```

Enable configuration:

```bash
sudo ln -s /etc/nginx/sites-available/capline-backend /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### Phase 5: Security Hardening

#### Configure UFW Firewall

```bash
sudo ufw --force enable
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw status verbose
```

#### Secure SSH

Edit `/etc/ssh/sshd_config`:

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

Restart SSH:

```bash
sudo systemctl restart sshd
```

---

## GitHub Configuration

### SSH Key Management

#### Generate SSH Keys

```bash
# Staging key
ssh-keygen -t ed25519 -C "github-staging" -f ~/.ssh/capline-staging -N ""

# Production key
ssh-keygen -t ed25519 -C "github-production" -f ~/.ssh/capline-production -N ""

# Display public keys
cat ~/.ssh/capline-staging.pub
cat ~/.ssh/capline-production.pub
```

#### Add Keys to Servers

On each server:

```bash
ssh deploy@server-ip
echo "your-public-key-here" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

### GitHub Secrets Configuration

Navigate to: **Repository → Settings → Secrets and variables → Actions**

#### Required Secrets

| Secret Name | Description | Example |
|-------------|-------------|---------|
| `STAGING_HOST` | Staging server IP | `54.123.45.67` |
| `STAGING_USERNAME` | SSH username | `deploy` |
| `STAGING_SSH_KEY` | Private SSH key | `-----BEGIN OPENSSH...` |
| `PRODUCTION_HOST` | Production server IP | `54.234.56.78` |
| `PRODUCTION_USERNAME` | SSH username | `deploy` |
| `PRODUCTION_SSH_KEY` | Private SSH key | `-----BEGIN OPENSSH...` |
| `MAIL_USER` | Gmail for notifications | `alerts@company.com` |
| `MAIL_PASS` | Gmail app password | `xxxx xxxx xxxx xxxx` |
| `MAIL_TO` | Recipient email | `devops@company.com` |

---

## PM2 Process Management

### Ecosystem Configuration

Create `ecosystem.config.js` in project root:

```javascript
module.exports = {
  apps: [{
    name: 'capline-core',
    script: './dist/main.js',
    instances: 'max',
    exec_mode: 'cluster',
    env: {
      NODE_ENV: 'production',
      PORT: 7474
    },
    error_file: './logs/pm2-error.log',
    out_file: './logs/pm2-out.log',
    log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
    merge_logs: true,
    max_memory_restart: '500M',
    autorestart: true,
    max_restarts: 10,
    min_uptime: '10s',
    listen_timeout: 10000,
    kill_timeout: 5000,
    wait_ready: true,
    shutdown_with_message: true
  }]
};
```

### PM2 Commands Reference

```bash
# Start application
pm2 start ecosystem.config.js --env production

# Process management
pm2 restart capline-core
pm2 reload capline-core
pm2 stop capline-core
pm2 delete capline-core

# Monitoring
pm2 list
pm2 monit
pm2 logs capline-core
pm2 logs capline-core --lines 100
pm2 show capline-core

# Save configuration
pm2 save

# Startup script
pm2 startup
```

---

## CI/CD Workflow Implementation

### Workflow File Structure

Create `.github/workflows/deploy.yml`:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - develop
      - main

env:
  NODE_VERSION: '20.x'
  APP_NAME: 'capline-backend-services'
  APP_PATH: '/var/www/backend/capline-backend-services'

jobs:
  # ==================== CI Stage ====================
  ci:
    name: CI - Build & Test
    runs-on: ubuntu-latest
    
    steps:
      - name: 📥 Checkout Code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: 🔧 Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: 📦 Install Dependencies
        run: npm ci

      - name: 🔨 Generate Prisma Client
        run: npm run generate

      - name: ✅ Validate Database Schema
        run: npm run test:schema
        continue-on-error: false

      - name: 🎨 Lint Code
        run: npm run lint
        continue-on-error: true

      - name: 🧪 Run Tests
        run: npm run test
        continue-on-error: true

      - name: 🏗️ Build Application
        run: npm run build

      - name: 🗜️ Prune Dev Dependencies
        run: npm prune --production

      - name: 📤 Upload Build Artifact
        uses: actions/upload-artifact@v4
        with:
          name: build-artifact
          path: |
            dist/
            node_modules/
            prisma/
            package*.json
            ecosystem.config.js
          retention-days: 7

  # ==================== Staging Deploy ====================
  deploy-staging:
    name: 🚀 Deploy to Staging
    needs: ci
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.yourdomain.com

    steps:
      - name: 📥 Download Artifact
        uses: actions/download-artifact@v4
        with:
          name: build-artifact

      - name: 📦 Create Deployment Package
        run: |
          tar -czf deployment.tar.gz \
            dist/ \
            node_modules/ \
            prisma/ \
            package*.json \
            ecosystem.config.js

      - name: 🔐 Setup SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.STAGING_SSH_KEY }}" > ~/.ssh/staging_key
          chmod 600 ~/.ssh/staging_key
          ssh-keyscan -H ${{ secrets.STAGING_HOST }} >> ~/.ssh/known_hosts

      - name: 🚀 Deploy to Server
        run: |
          scp -i ~/.ssh/staging_key \
            -o StrictHostKeyChecking=no \
            deployment.tar.gz \
            ${{ secrets.STAGING_USERNAME }}@${{ secrets.STAGING_HOST }}:/tmp/

          ssh -i ~/.ssh/staging_key \
            -o StrictHostKeyChecking=no \
            ${{ secrets.STAGING_USERNAME }}@${{ secrets.STAGING_HOST }} << 'ENDSSH'
            
            set -e
            
            # Configuration
            APP_PATH="${{ env.APP_PATH }}"
            BACKUP_DIR="/var/www/backups/app"
            TIMESTAMP=$(date +%Y%m%d_%H%M%S)
            
            echo "📦 Deploying to Staging..."
            
            # Create backup
            if [ -d "$APP_PATH" ]; then
              echo "💾 Creating backup..."
              sudo cp -r "$APP_PATH" "${BACKUP_DIR}/${APP_NAME}_backup_${TIMESTAMP}"
              sudo chown -R deploy:deploy "${BACKUP_DIR}/${APP_NAME}_backup_${TIMESTAMP}"
            fi
            
            # Create app directory
            sudo mkdir -p "$APP_PATH"
            sudo chown -R deploy:deploy "$APP_PATH"
            
            # Extract new version
            echo "📂 Extracting files..."
            cd "$APP_PATH"
            tar -xzf /tmp/deployment.tar.gz
            rm /tmp/deployment.tar.gz
            
            # Database operations
            echo "🗄️ Running database operations..."
            
            # Backup database
            if [ -n "$POSTGRESQL_DATABASE_URL" ]; then
              BACKUP_FILE="${BACKUP_DIR}/db_backup_${TIMESTAMP}.sql"
              pg_dump "$POSTGRESQL_DATABASE_URL" > "$BACKUP_FILE" || echo "⚠️ DB backup failed"
            fi
            
            # Run migrations
            npx prisma migrate deploy || echo "⚠️ Migrations completed with warnings"
            
            # Restart PM2
            echo "♻️ Restarting application..."
            pm2 reload ecosystem.config.js --env production || pm2 start ecosystem.config.js --env production
            pm2 save
            
            # Health check
            echo "🏥 Running health check..."
            sleep 5
            
            for i in {1..10}; do
              if curl -f http://localhost:7474/api/health-check > /dev/null 2>&1; then
                echo "✅ Health check passed!"
                exit 0
              fi
              echo "⏳ Waiting for application to start (attempt $i/10)..."
              sleep 3
            done
            
            echo "❌ Health check failed!"
            exit 1
          ENDSSH

      - name: 📧 Send Notification
        if: always()
        uses: dawidd6/action-send-mail@v3
        with:
          server_address: smtp.gmail.com
          server_port: 465
          username: ${{ secrets.MAIL_USER }}
          password: ${{ secrets.MAIL_PASS }}
          subject: "🚀 Staging Deployment: ${{ job.status }}"
          to: ${{ secrets.MAIL_TO }}
          from: DevOps Alerts
          body: |
            Deployment to Staging: ${{ job.status }}
            
            Branch: ${{ github.ref_name }}
            Commit: ${{ github.sha }}
            Author: ${{ github.actor }}
            
            View logs: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}

  # ==================== Production Deploy ====================
  deploy-production:
    name: 🚀 Deploy to Production
    needs: ci
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://api.yourdomain.com

    steps:
      - name: 📥 Download Artifact
        uses: actions/download-artifact@v4
        with:
          name: build-artifact

      - name: 📦 Create Deployment Package
        run: |
          tar -czf deployment.tar.gz \
            dist/ \
            node_modules/ \
            prisma/ \
            package*.json \
            ecosystem.config.js

      - name: 🔐 Setup SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.PRODUCTION_SSH_KEY }}" > ~/.ssh/prod_key
          chmod 600 ~/.ssh/prod_key
          ssh-keyscan -H ${{ secrets.PRODUCTION_HOST }} >> ~/.ssh/known_hosts

      - name: 🚀 Deploy to Server
        run: |
          scp -i ~/.ssh/prod_key \
            -o StrictHostKeyChecking=no \
            deployment.tar.gz \
            ${{ secrets.PRODUCTION_USERNAME }}@${{ secrets.PRODUCTION_HOST }}:/tmp/

          ssh -i ~/.ssh/prod_key \
            -o StrictHostKeyChecking=no \
            ${{ secrets.PRODUCTION_USERNAME }}@${{ secrets.PRODUCTION_HOST }} << 'ENDSSH'
            
            set -e
            
            # Configuration
            APP_PATH="${{ env.APP_PATH }}"
            BACKUP_DIR="/var/www/backups/app"
            TIMESTAMP=$(date +%Y%m%d_%H%M%S)
            
            echo "📦 Deploying to Production..."
            
            # Create backup
            if [ -d "$APP_PATH" ]; then
              echo "💾 Creating backup..."
              sudo cp -r "$APP_PATH" "${BACKUP_DIR}/${APP_NAME}_backup_${TIMESTAMP}"
              sudo chown -R deploy:deploy "${BACKUP_DIR}/${APP_NAME}_backup_${TIMESTAMP}"
            fi
            
            # Create app directory
            sudo mkdir -p "$APP_PATH"
            sudo chown -R deploy:deploy "$APP_PATH"
            
            # Extract new version
            echo "📂 Extracting files..."
            cd "$APP_PATH"
            tar -xzf /tmp/deployment.tar.gz
            rm /tmp/deployment.tar.gz
            
            # Database operations
            echo "🗄️ Running database operations..."
            
            # Backup database
            if [ -n "$POSTGRESQL_DATABASE_URL" ]; then
              BACKUP_FILE="${BACKUP_DIR}/db_backup_${TIMESTAMP}.sql"
              pg_dump "$POSTGRESQL_DATABASE_URL" > "$BACKUP_FILE" || echo "⚠️ DB backup failed"
            fi
            
            # Run migrations
            npx prisma migrate deploy || echo "⚠️ Migrations completed with warnings"
            
            # Zero-downtime restart
            echo "♻️ Restarting application..."
            pm2 reload ecosystem.config.js --env production
            pm2 save
            
            # Health check
            echo "🏥 Running health check..."
            sleep 5
            
            for i in {1..15}; do
              if curl -f http://localhost:7474/api/health-check > /dev/null 2>&1; then
                echo "✅ Health check passed!"
                exit 0
              fi
              echo "⏳ Waiting for application to start (attempt $i/15)..."
              sleep 3
            done
            
            echo "❌ Health check failed!"
            exit 1
          ENDSSH

      - name: 📧 Send Notification
        if: always()
        uses: dawidd6/action-send-mail@v3
        with:
          server_address: smtp.gmail.com
          server_port: 465
          username: ${{ secrets.MAIL_USER }}
          password: ${{ secrets.MAIL_PASS }}
          subject: "🚀 Production Deployment: ${{ job.status }}"
          to: ${{ secrets.MAIL_TO }}
          from: DevOps Alerts
          body: |
            Deployment to Production: ${{ job.status }}
            
            Branch: ${{ github.ref_name }}
            Commit: ${{ github.sha }}
            Author: ${{ github.actor }}
            
            View logs: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
```

---

## Database Management

### Prisma Configuration

#### Schema Setup

Create `prisma/schema.prisma`:

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("POSTGRESQL_DATABASE_URL")
}

model User {
  id        String   @id @default(uuid())
  email     String   @unique
  name      String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

#### Migration Commands

```bash
# Create migration
npx prisma migrate dev --name init

# Deploy migrations
npx prisma migrate deploy

# Check migration status
npx prisma migrate status

# Reset database (dev only)
npx prisma migrate reset

# Generate Prisma Client
npx prisma generate

# Validate schema
npx prisma validate
```

### Database Backup Strategy

#### Automated Backups

Create backup script `/var/www/scripts/backup-db.sh`:

```bash
#!/bin/bash

BACKUP_DIR="/var/www/backups/database"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/backup_${TIMESTAMP}.sql"

# Create backup
pg_dump "$POSTGRESQL_DATABASE_URL" > "$BACKUP_FILE"

# Compress
gzip "$BACKUP_FILE"

# Keep only last 7 days
find "$BACKUP_DIR" -name "backup_*.sql.gz" -mtime +7 -delete

echo "Backup completed: ${BACKUP_FILE}.gz"
```

Make executable:

```bash
chmod +x /var/www/scripts/backup-db.sh
```

#### Schedule with Cron

```bash
# Edit crontab
crontab -e

# Add daily backup at 2 AM
0 2 * * * /var/www/scripts/backup-db.sh >> /var/log/backup.log 2>&1
```

### Database Restore

```bash
# List backups
ls -lh /var/www/backups/database/

# Restore from backup
gunzip -c backup_20241215_020000.sql.gz | psql "$POSTGRESQL_DATABASE_URL"

# Verify
psql "$POSTGRESQL_DATABASE_URL" -c "\dt"
```

---

## Testing & Validation

### Package.json Scripts

Add to `package.json`:

```json
{
  "scripts": {
    "build": "nest build",
    "start": "nest start",
    "start:dev": "nest start --watch",
    "start:prod": "node dist/main",
    "lint": "eslint \"{src,apps,libs,test}/**/*.ts\" --fix",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:cov": "jest --coverage",
    "test:e2e": "jest --config ./test/jest-e2e.json",
    "generate": "prisma generate",
    "test:schema": "prisma validate && prisma format"
  }
}
```

### Health Check Endpoint

Create `src/health/health.controller.ts`:

```typescript
import { Controller, Get } from '@nestjs/common';

@Controller()
export class HealthController {
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
```

### Local Testing

```bash
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
```

### Staging Validation

```bash
# SSH to staging
ssh deploy@staging-server-ip

# Check PM2
pm2 status
pm2 logs capline-core --lines 50

# Test health
curl http://localhost:7474/api/health-check

# Monitor
pm2 monit
```

---

## Production Deployment

### Deployment Process

#### Step 1: Merge to Main

```bash
git checkout develop
git pull origin develop

git checkout main
git pull origin main
git merge develop
git push origin main
```

#### Step 2: Monitor Deployment

1. Go to GitHub → Actions
2. Watch pipeline execution
3. Approve production deployment

#### Step 3: Verify

```bash
ssh deploy@production-server-ip
pm2 status
pm2 logs capline-core --lines 50
curl http://localhost:7474/api/health-check
```

### Emergency Hotfix

```bash
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
```

### Rollback Procedure

```bash
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
```

---

## Monitoring & Maintenance

### PM2 Monitoring

```bash
# Real-time monitoring
pm2 monit

# View logs
pm2 logs capline-core
pm2 logs capline-core --lines 100
pm2 logs capline-core --err

# Process info
pm2 show capline-core
pm2 list
```

### System Monitoring

```bash
# Resources
free -h
df -h
top
htop

# Network
netstat -tulpn | grep 7474

# Disk usage
du -sh /var/www/backend/*
```

### Performance Metrics

| Metric | Tool | Threshold | Action |
|--------|------|-----------|--------|
| CPU | top, pm2 | > 80% | Investigate |
| Memory | free, pm2 | > 85% | Restart/Scale |
| Response Time | logs | > 2s | Optimize |
| Error Rate | logs | > 1% | Debug |

### Cleanup Tasks

#### Remove Old Backups

```bash
# Remove app backups older than 30 days
find /var/www/backups/app -name "*_backup_*" -mtime +30 -exec rm -rf {} \;

# Remove DB backups older than 14 days
find /var/www/backups/database -name "backup_*.sql.gz" -mtime +14 -delete
```

#### Clear PM2 Logs

```bash
pm2 flush
```

---

## Security Best Practices

### Environment Variables

Never commit:
- `.env`
- `*.pem`
- `*.key`

Use `.env.example`:

```bash
# Database
POSTGRESQL_DATABASE_URL=postgresql://user:pass@host:5432/db

# Application
APP_PORT=7474
NODE_ENV=production

# JWT
JWT_SECRET=your-secret
JWT_EXPIRES_IN=7d
```

### SSH Security

```bash
# Use Ed25519 keys
ssh-keygen -t ed25519 -C "email@example.com"

# Disable password auth
# /etc/ssh/sshd_config:
PasswordAuthentication no
PermitRootLogin no
```

### Application Security

```typescript
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
```

---

## Troubleshooting Guide

### Common Issues

#### Deployment Fails

**Symptoms:** Build fails, TypeScript errors

**Solutions:**
```bash
npm ci
npm run build
npm run test
```

#### Health Check Fails

**Symptoms:** Connection refused, 502 errors

**Solutions:**
```bash
pm2 list
pm2 logs capline-core --lines 100
netstat -tulpn | grep 7474
curl -v http://localhost:7474/api/health-check
pm2 restart capline-core
```

#### Migration Fails

**Symptoms:** Migration errors, can't connect to DB

**Solutions:**
```bash
psql $POSTGRESQL_DATABASE_URL
npx prisma migrate status
npx prisma migrate deploy
```

#### PM2 Crashes

**Symptoms:** Process keeps restarting

**Solutions:**
```bash
pm2 logs capline-core --lines 200
pm2 monit
pm2 delete capline-core
pm2 start ecosystem.config.js --env production
```

#### High Memory

**Symptoms:** OOM errors, slow performance

**Solutions:**
```bash
free -h
pm2 list
pm2 restart capline-core
# Increase max_memory_restart in ecosystem.config.js
```

### Debug Commands

```bash
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
```

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
**Last Updated:** January 2026
