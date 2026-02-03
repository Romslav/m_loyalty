# 🚀 DEPLOYMENT GUIDE - MAX-LOYALTY

## 📋 СОДЕРЖАНИЕ

1. [Требования](#требования)
2. [Local Development](#local-development)
3. [Docker Deployment](#docker-deployment)
4. [Production Deployment](#production-deployment)
5. [Настройка переменных окружения](#настройка-переменных-окружения)
6. [Database Migration](#database-migration)
7. [CI/CD Pipeline](#cicd-pipeline)
8. [Monitoring & Logging](#monitoring--logging)
9. [Бэкап и восстановление](#бэкап-и-восстановление)

---

## ТРЕБОВАНИЯ

### System Requirements

#### Development
- **OS:** Linux, macOS, Windows (WSL2)
- **Node.js:** 20.x LTS
- **pnpm:** 8.x+
- **Docker:** 24.x+
- **Docker Compose:** 2.20+
- **PostgreSQL:** 15+
- **Redis:** 7+

#### Production
- **CPU:** 4+ cores
- **RAM:** 8GB+ (минимум 16GB рекомендуется)
- **Disk:** 50GB+ SSD
- **Network:** 100 Mbps+

### Software Dependencies

```json
{
  "backend": {
    "node": "20.x",
    "nestjs": "10.x",
    "prisma": "5.x",
    "typescript": "5.x"
  },
  "frontend": {
    "react": "18.x",
    "vite": "5.x",
    "typescript": "5.x"
  },
  "telegram-bot": {
    "telegraf": "4.x",
    "node": "20.x"
  },
  "database": {
    "postgresql": "15+",
    "redis": "7+"
  }
}
```

---

## LOCAL DEVELOPMENT

### 1. Clone Repository

```bash
git clone https://github.com/Romslav/m_loyalty.git
cd m_loyalty
```

### 2. Install Dependencies

```bash
# Install pnpm globally
npm install -g pnpm

# Install all workspace dependencies
pnpm install
```

### 3. Setup Database

#### Using Docker Compose

```bash
# Start PostgreSQL and Redis
docker-compose -f docker-compose.dev.yml up -d postgres redis

# Wait for services to be ready
docker-compose -f docker-compose.dev.yml ps
```

#### Manual Setup

```bash
# Create PostgreSQL database
createdb maxloyalty_dev

# Start Redis
redis-server
```

### 4. Environment Configuration

```bash
# Copy environment template
cp .env.example .env

# Edit .env with your local settings
vim .env
```

**Minimal `.env` for development:**

```env
# Database
DATABASE_URL="postgresql://postgres:password@localhost:5432/maxloyalty_dev"

# Redis
REDIS_URL="redis://localhost:6379"

# JWT
JWT_SECRET="your-development-secret-key-here"
JWT_REFRESH_SECRET="your-development-refresh-secret-here"

# Telegram
TELEGRAM_BOT_TOKEN="your-telegram-bot-token"
TELEGRAM_MINI_APP_URL="http://localhost:5173"

# Frontend
VITE_API_URL="http://localhost:3000"
```

### 5. Database Migration

```bash
# Generate Prisma client
pnpm prisma:generate

# Run migrations
pnpm prisma:migrate:dev

# Seed database
pnpm prisma:seed
```

### 6. Start Development Servers

#### Option A: Start all services

```bash
pnpm dev
```

This will start:
- Backend API: http://localhost:3000
- Frontend: http://localhost:5173
- Telegram Bot: Running in background

#### Option B: Start services individually

```bash
# Terminal 1: Backend
pnpm --filter backend dev

# Terminal 2: Frontend
pnpm --filter frontend dev

# Terminal 3: Telegram Bot
pnpm --filter telegram-bot dev
```

### 7. Verify Installation

```bash
# Check backend health
curl http://localhost:3000/health

# Check API docs
open http://localhost:3000/docs

# Check frontend
open http://localhost:5173
```

---

## DOCKER DEPLOYMENT

### Development with Docker

```bash
# Build all images
docker-compose -f docker-compose.dev.yml build

# Start all services
docker-compose -f docker-compose.dev.yml up -d

# View logs
docker-compose -f docker-compose.dev.yml logs -f

# Stop services
docker-compose -f docker-compose.dev.yml down
```

### Production with Docker

```bash
# Build production images
docker-compose -f docker-compose.prod.yml build

# Start services
docker-compose -f docker-compose.prod.yml up -d

# Check status
docker-compose -f docker-compose.prod.yml ps
```

### docker-compose.prod.yml

```yaml
version: '3.8'

services:
  backend:
    build:
      context: ./apps/backend
      dockerfile: Dockerfile.prod
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: production
      DATABASE_URL: ${DATABASE_URL}
      REDIS_URL: ${REDIS_URL}
      JWT_SECRET: ${JWT_SECRET}
    depends_on:
      - postgres
      - redis
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  frontend:
    build:
      context: ./apps/frontend
      dockerfile: Dockerfile.prod
    ports:
      - "80:80"
    depends_on:
      - backend
    restart: unless-stopped

  telegram-bot:
    build:
      context: ./apps/telegram-bot
      dockerfile: Dockerfile.prod
    environment:
      TELEGRAM_BOT_TOKEN: ${TELEGRAM_BOT_TOKEN}
      API_URL: http://backend:3000
    depends_on:
      - backend
    restart: unless-stopped

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  nginx:
    image: nginx:alpine
    ports:
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      - backend
      - frontend
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:

networks:
  default:
    name: maxloyalty_network
```

---

## PRODUCTION DEPLOYMENT

### Option 1: VPS Deployment (DigitalOcean, AWS, etc.)

#### 1. Server Setup

```bash
# Connect to server
ssh root@your-server-ip

# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Install Docker Compose
sudo apt install docker-compose-plugin

# Install nginx
sudo apt install nginx certbot python3-certbot-nginx
```

#### 2. Clone & Configure

```bash
# Clone repository
git clone https://github.com/Romslav/m_loyalty.git
cd m_loyalty

# Create production .env
cp .env.production.example .env.production
vim .env.production
```

#### 3. SSL Certificate

```bash
# Obtain SSL certificate
sudo certbot --nginx -d api.max-loyalty.com -d max-loyalty.com

# Auto-renewal
sudo certbot renew --dry-run
```

#### 4. Deploy

```bash
# Build and start
docker-compose -f docker-compose.prod.yml up -d --build

# Run migrations
docker-compose -f docker-compose.prod.yml exec backend pnpm prisma:migrate:deploy

# Seed initial data
docker-compose -f docker-compose.prod.yml exec backend pnpm prisma:seed
```

#### 5. Verify

```bash
# Check services
docker-compose -f docker-compose.prod.yml ps

# View logs
docker-compose -f docker-compose.prod.yml logs -f backend

# Test API
curl https://api.max-loyalty.com/health
```

### Option 2: Kubernetes Deployment

#### Prerequisites
- Kubernetes cluster (GKE, EKS, AKS, or self-hosted)
- kubectl configured
- Helm 3.x installed

#### 1. Create Namespace

```bash
kubectl create namespace maxloyalty
```

#### 2. Create Secrets

```bash
kubectl create secret generic maxloyalty-secrets \
  --from-literal=database-url="postgresql://..." \
  --from-literal=jwt-secret="..." \
  --from-literal=telegram-bot-token="..." \
  -n maxloyalty
```

#### 3. Deploy with Helm

```bash
# Add Helm repository (if using)
helm repo add maxloyalty https://charts.max-loyalty.com
helm repo update

# Install
helm install maxloyalty maxloyalty/maxloyalty \
  --namespace maxloyalty \
  --values values.production.yaml
```

#### 4. Verify Deployment

```bash
# Check pods
kubectl get pods -n maxloyalty

# Check services
kubectl get svc -n maxloyalty

# View logs
kubectl logs -f deployment/maxloyalty-backend -n maxloyalty
```

---

## НАСТРОЙКА ПЕРЕМЕННЫХ ОКРУЖЕНИЯ

### Production Environment Variables

```env
# ==========================================
# APPLICATION
# ==========================================
NODE_ENV=production
PORT=3000
API_URL=https://api.max-loyalty.com
FRONTEND_URL=https://max-loyalty.com

# ==========================================
# DATABASE
# ==========================================
DATABASE_URL="postgresql://user:password@postgres:5432/maxloyalty?schema=public&sslmode=require"
DATABASE_POOL_MIN=2
DATABASE_POOL_MAX=10
DATABASE_CONNECTION_TIMEOUT=20000

# ==========================================
# REDIS
# ==========================================
REDIS_URL="redis://:password@redis:6379"
REDIS_TTL=3600
REDIS_MAX_RETRIES=3

# ==========================================
# JWT
# ==========================================
JWT_SECRET="your-super-secret-jwt-key-min-32-chars"
JWT_REFRESH_SECRET="your-super-secret-refresh-key-min-32-chars"
JWT_ACCESS_TOKEN_TTL=900          # 15 minutes
JWT_REFRESH_TOKEN_TTL=2592000    # 30 days

# ==========================================
# TELEGRAM
# ==========================================
TELEGRAM_BOT_TOKEN="your-telegram-bot-token"
TELEGRAM_BOT_USERNAME="your_loyalty_bot"
TELEGRAM_MINI_APP_URL="https://max-loyalty.com/mini-app"
TELEGRAM_WEBHOOK_URL="https://api.max-loyalty.com/telegram/webhook"

# ==========================================
# PAYMENT GATEWAYS
# ==========================================
# Stripe
STRIPE_SECRET_KEY="sk_live_..."
STRIPE_WEBHOOK_SECRET="whsec_..."

# YooKassa
YOOKASSA_SHOP_ID="your-shop-id"
YOOKASSA_SECRET_KEY="your-secret-key"

# ==========================================
# POS INTEGRATIONS
# ==========================================
# iiko
IIKO_API_URL="https://api.iiko.ru"
IIKO_WEBHOOK_SECRET="your-webhook-secret"

# R-Keeper
RKEEPER_API_URL="https://api.rkeeper.com"
RKEEPER_WEBHOOK_SECRET="your-webhook-secret"

# ==========================================
# EMAIL
# ==========================================
SMTP_HOST="smtp.gmail.com"
SMTP_PORT=587
SMTP_SECURE=false
SMTP_USER="noreply@max-loyalty.com"
SMTP_PASSWORD="your-email-password"
SMTP_FROM="MAX-LOYALTY <noreply@max-loyalty.com>"

# ==========================================
# SMS
# ==========================================
SMS_PROVIDER="twilio"  # twilio, vonage, smsru
TWILIO_ACCOUNT_SID="your-account-sid"
TWILIO_AUTH_TOKEN="your-auth-token"
TWILIO_PHONE_NUMBER="+1234567890"

# ==========================================
# FILE STORAGE
# ==========================================
STORAGE_TYPE="s3"  # local, s3, gcs
AWS_ACCESS_KEY_ID="your-access-key"
AWS_SECRET_ACCESS_KEY="your-secret-key"
AWS_S3_BUCKET="maxloyalty-uploads"
AWS_S3_REGION="eu-central-1"

# ==========================================
# MONITORING & LOGGING
# ==========================================
LOG_LEVEL="info"  # debug, info, warn, error
SENTRY_DSN="https://...@sentry.io/..."

# Prometheus
METRICS_ENABLED=true
METRICS_PORT=9090

# ==========================================
# RATE LIMITING
# ==========================================
RATE_LIMIT_TTL=60
RATE_LIMIT_MAX=1000

# ==========================================
# CORS
# ==========================================
CORS_ORIGIN="https://max-loyalty.com,https://admin.max-loyalty.com"
CORS_CREDENTIALS=true

# ==========================================
# SECURITY
# ==========================================
HELMET_ENABLED=true
CSRF_ENABLED=true
CSRF_SECRET="your-csrf-secret"

# ==========================================
# BULL QUEUE
# ==========================================
BULL_BOARD_ENABLED=false
BULL_BOARD_USERNAME="admin"
BULL_BOARD_PASSWORD="secure-password"
```

---

## DATABASE MIGRATION

### Development Migrations

```bash
# Create new migration
pnpm prisma:migrate:dev --name add_new_feature

# Generate Prisma client
pnpm prisma:generate

# Reset database (WARNING: deletes all data)
pnpm prisma:migrate:reset
```

### Production Migrations

```bash
# 1. Backup database first!
pg_dump -U postgres maxloyalty > backup_$(date +%Y%m%d_%H%M%S).sql

# 2. Deploy migrations
pnpm prisma:migrate:deploy

# 3. Verify
pnpm prisma:db:seed  # If needed
```

### Migration Best Practices

1. **Always backup before migrations**
2. **Test migrations on staging first**
3. **Use transactions for data migrations**
4. **Monitor migration execution time**
5. **Have rollback plan ready**

### Rollback Procedure

```bash
# 1. Stop application
docker-compose -f docker-compose.prod.yml stop backend

# 2. Restore database
psql -U postgres maxloyalty < backup_20260203_120000.sql

# 3. Revert to previous version
git checkout <previous-commit>
docker-compose -f docker-compose.prod.yml up -d --build
```

---

## CI/CD PIPELINE

### GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: maxloyalty_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'
      
      - name: Install pnpm
        run: npm install -g pnpm
      
      - name: Install dependencies
        run: pnpm install
      
      - name: Generate Prisma Client
        run: pnpm prisma:generate
      
      - name: Run migrations
        run: pnpm prisma:migrate:deploy
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/maxloyalty_test
      
      - name: Run unit tests
        run: pnpm test
      
      - name: Run integration tests
        run: pnpm test:e2e
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/maxloyalty_test
          REDIS_URL: redis://localhost:6379
  
  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Login to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      
      - name: Build and push Backend
        uses: docker/build-push-action@v4
        with:
          context: ./apps/backend
          push: true
          tags: |
            maxloyalty/backend:latest
            maxloyalty/backend:${{ github.sha }}
      
      - name: Build and push Frontend
        uses: docker/build-push-action@v4
        with:
          context: ./apps/frontend
          push: true
          tags: |
            maxloyalty/frontend:latest
            maxloyalty/frontend:${{ github.sha }}
      
      - name: Build and push Telegram Bot
        uses: docker/build-push-action@v4
        with:
          context: ./apps/telegram-bot
          push: true
          tags: |
            maxloyalty/telegram-bot:latest
            maxloyalty/telegram-bot:${{ github.sha }}
  
  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to server
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /opt/maxloyalty
            git pull origin main
            docker-compose -f docker-compose.prod.yml pull
            docker-compose -f docker-compose.prod.yml up -d --build
            docker-compose -f docker-compose.prod.yml exec -T backend pnpm prisma:migrate:deploy
      
      - name: Notify Slack
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: 'Deployment to production completed'
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

---

## MONITORING & LOGGING

### Prometheus + Grafana

#### docker-compose.monitoring.yml

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    restart: unless-stopped
  
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
      GF_INSTALL_PLUGINS: grafana-piechart-panel
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    restart: unless-stopped
  
  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    volumes:
      - ./loki/loki-config.yml:/etc/loki/local-config.yaml
      - loki_data:/loki
    command: -config.file=/etc/loki/local-config.yaml
    restart: unless-stopped
  
  promtail:
    image: grafana/promtail:latest
    volumes:
      - ./promtail/promtail-config.yml:/etc/promtail/config.yml
      - /var/log:/var/log
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
    command: -config.file=/etc/promtail/config.yml
    restart: unless-stopped

volumes:
  prometheus_data:
  grafana_data:
  loki_data:
```

### Application Logs

```typescript
// logger.config.ts
import { WinstonModule } from 'nest-winston';
import * as winston from 'winston';

export const loggerConfig = WinstonModule.createLogger({
  transports: [
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.colorize(),
        winston.format.printf(({ timestamp, level, message, context }) => {
          return `${timestamp} [${context}] ${level}: ${message}`;
        }),
      ),
    }),
    new winston.transports.File({
      filename: 'logs/error.log',
      level: 'error',
      format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json(),
      ),
    }),
    new winston.transports.File({
      filename: 'logs/combined.log',
      format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json(),
      ),
    }),
  ],
});
```

---

## БЭКАП И ВОССТАНОВЛЕНИЕ

### Automated Backup Script

```bash
#!/bin/bash
# backup.sh

BACKUP_DIR="/backups/maxloyalty"
DATE=$(date +%Y%m%d_%H%M%S)
DB_NAME="maxloyalty"
DB_USER="postgres"

# Create backup directory
mkdir -p $BACKUP_DIR

# Backup PostgreSQL
echo "Creating database backup..."
pg_dump -U $DB_USER -Fc $DB_NAME > $BACKUP_DIR/db_$DATE.dump

# Backup Redis (if needed)
echo "Creating Redis backup..."
redis-cli --rdb $BACKUP_DIR/redis_$DATE.rdb

# Backup files
echo "Creating files backup..."
tar -czf $BACKUP_DIR/files_$DATE.tar.gz /opt/maxloyalty/uploads

# Remove backups older than 30 days
find $BACKUP_DIR -name "*.dump" -mtime +30 -delete
find $BACKUP_DIR -name "*.rdb" -mtime +30 -delete
find $BACKUP_DIR -name "*.tar.gz" -mtime +30 -delete

echo "Backup completed: $DATE"
```

### Setup Cron Job

```bash
# Add to crontab
crontab -e

# Run backup daily at 2 AM
0 2 * * * /opt/maxloyalty/scripts/backup.sh >> /var/log/maxloyalty-backup.log 2>&1
```

### Restore Procedure

```bash
#!/bin/bash
# restore.sh

BACKUP_FILE=$1

if [ -z "$BACKUP_FILE" ]; then
    echo "Usage: ./restore.sh <backup_file>"
    exit 1
fi

echo "Restoring database from $BACKUP_FILE..."

# Stop application
docker-compose -f docker-compose.prod.yml stop backend

# Drop existing database
psql -U postgres -c "DROP DATABASE IF EXISTS maxloyalty;"
psql -U postgres -c "CREATE DATABASE maxloyalty;"

# Restore database
pg_restore -U postgres -d maxloyalty $BACKUP_FILE

# Start application
docker-compose -f docker-compose.prod.yml up -d backend

echo "Restore completed!"
```

---

## 🛠️ TROUBLESHOOTING

### Common Issues

#### Database Connection Failed

```bash
# Check PostgreSQL is running
docker-compose ps postgres

# Check connection
psql -U postgres -h localhost -p 5432 maxloyalty

# View logs
docker-compose logs postgres
```

#### Redis Connection Failed

```bash
# Check Redis is running
docker-compose ps redis

# Test connection
redis-cli ping

# View logs
docker-compose logs redis
```

#### Migration Errors

```bash
# Reset migration history
pnpm prisma:migrate:reset

# Force resolve
pnpm prisma:migrate:resolve --applied "migration_name"
```

#### High Memory Usage

```bash
# Check container stats
docker stats

# Restart services
docker-compose restart

# Clear Redis cache
redis-cli FLUSHALL
```

---

## 📚 USEFUL COMMANDS

```bash
# View all logs
docker-compose logs -f

# View specific service logs
docker-compose logs -f backend

# Enter container shell
docker-compose exec backend sh

# Database console
docker-compose exec postgres psql -U postgres maxloyalty

# Redis console
docker-compose exec redis redis-cli

# Check disk usage
df -h
du -sh /var/lib/docker

# Clean Docker resources
docker system prune -a --volumes

# Update services
git pull
docker-compose pull
docker-compose up -d --build
```

---

**Версия:** 1.0  
**Дата:** 2026-02-03  
**Статус:** Готово к использованию
