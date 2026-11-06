# Dongle Backend — Deployment Guide

## Deployment Status

⚠️ **NOT YET READY FOR DEPLOYMENT** — The backend is currently in alpha development. This guide documents the deployment architecture and process to follow once implementation is complete.

**Deployment timeline**: TBD (when core services are production-ready)

## Deployment Architecture

### Local Development

```
Developer Machine
├── Node.js 20.x (npm run dev)
├── PostgreSQL 14 (localhost:5432)
├── Soroban testnet RPC (remote)
└── .env.local (local config)
```

**Setup**:
```bash
# Install dependencies
pnpm install

# Initialize database
pnpm run db:migrate

# Start dev server
pnpm run dev

# Server running at http://localhost:3001
```

### Staging Environment

```
Staging Server (Cloud Provider)
├── Node.js 20.x (PM2 or systemd)
├── PostgreSQL RDS (managed service)
├── Redis cache (optional)
├── Soroban testnet RPC
├── SSL/TLS (staging certificate)
└── Monitoring (Sentry, Datadog)
```

**Deployment method**: GitHub Actions → Cloud storage → SSH deploy

### Production Environment

```
Production Servers (Load Balanced)
├── API Tier (3+ servers behind load balancer)
│   ├── Node.js 20.x (PM2)
│   ├── Auto-scaling based on CPU/memory
│   └── Health checks (liveness + readiness)
├── Data Tier
│   ├── PostgreSQL RDS (Multi-AZ, automated backups)
│   ├── Read replicas for analytics queries
│   └── Connection pooling (PgBouncer)
├── Cache Tier (optional)
│   ├── Redis cluster (for distributed caching)
│   └── Failover + replication
├── CDN (optional)
│   ├── Cloudfront / Cloudflare
│   └── API rate limiting at edge
├── Monitoring & Logging
│   ├── Sentry (error tracking)
│   ├── Datadog / ELK (logs and metrics)
│   ├── CloudWatch alarms
│   └── Status page (Statuspage.io)
└── DNS & SSL
    ├── Route 53 / managed DNS
    └── ACM / Let's Encrypt (SSL certificates)
```

## Prerequisites for Deployment

### Required Infrastructure

- [ ] Cloud hosting account (AWS, GCP, Azure, or similar)
- [ ] PostgreSQL database (managed service recommended)
- [ ] Domain name with DNS access
- [ ] SSL/TLS certificate (Let's Encrypt or ACM)
- [ ] Monitoring and logging service (Sentry, Datadog)
- [ ] CI/CD pipeline (GitHub Actions, GitLab CI)

### Required Credentials

- [ ] Stellar mainnet account with XLM for contract deployment
- [ ] Smart contract IDs deployed to Soroban
- [ ] JWT secret (min 32 characters, unique per environment)
- [ ] Database credentials with full permissions
- [ ] Admin wallet addresses for ADMIN_ALLOWLIST

### Required Configuration

- [ ] Environment variables validated in `.env`
- [ ] Smart contract IDs for mainnet
- [ ] CORS domain configured for frontend
- [ ] Database backups configured
- [ ] Log retention policy set

## Deployment Checklist

### Pre-Deployment (1 week before)

- [ ] Code frozen for testing
- [ ] Staging environment tested thoroughly
- [ ] Database migration tested with production data snapshot
- [ ] Disaster recovery procedure tested
- [ ] Load testing completed (target: 1000 req/s)
- [ ] Security audit completed
- [ ] Monitoring and alerting configured
- [ ] Team training on monitoring dashboard
- [ ] Deployment runbook prepared

### Environment Configuration

Create `.env.production` (secure, never commit):

```env
# Server
NODE_ENV=production
PORT=3001
LOG_LEVEL=info

# Database (production RDS)
DATABASE_URL=postgresql://user:password@prod-db.example.com:5432/dongle_prod
DATABASE_POOL_MIN=10
DATABASE_POOL_MAX=50

# Stellar/Soroban (mainnet)
STELLAR_NETWORK_PASSPHRASE=Public Global Stellar Network ; September 2015
SOROBAN_RPC_URL=https://soroban-mainnet.stellar.org

# Smart Contracts (deployed to mainnet)
PROJECT_REGISTRY_CONTRACT=CAE7WTI3X5HZQCDT5O3EH7FVDX7VKJP3WVLVZM3NUJSQFXLW5UQ4SVQ
REVIEW_REGISTRY_CONTRACT=CAE7WTI3X5HZQCDT5O3EH7FVDX7VKJP3WVLVZM3NUJSQFXLW5UQ4SVQ
VERIFICATION_REGISTRY_CONTRACT=CAE7WTI3X5HZQCDT5O3EH7FVDX7VKJP3WVLVZM3NUJSQFXLW5UQ4SVQ

# Security
JWT_SECRET=<generate-32-character-random-string>
ADMIN_ALLOWLIST=GXXXXX1,GXXXXX2,GXXXXX3

# Frontend
FRONTEND_URL=https://dongle.app

# Monitoring
SENTRY_DSN=https://...@sentry.io/...
DATADOG_API_KEY=...

# Optional: Caching
REDIS_URL=redis://redis-cluster.example.com:6379
```

### Deployment Steps

#### 1. Database Migration

```bash
# Run migrations on production database
# (this should be idempotent and reversible)
pnpm run db:migrate

# Verify migration success
pnpm run db:status

# If needed, rollback last migration
pnpm run db:rollback
```

#### 2. Build & Deploy

```bash
# Build production bundle
pnpm run build

# Run production build tests
pnpm run test

# Push to registry (Docker)
docker build -t dongle-backend:1.0.0 .
docker push registry.example.com/dongle-backend:1.0.0

# Deploy to Kubernetes / Cloud Run / ECS
# (depends on your platform)
```

#### 3. Smoke Tests

```bash
# Health check
curl https://api.dongle.app/health

# Readiness check
curl https://api.dongle.app/health/ready

# Sample API call
curl https://api.dongle.app/api/projects?limit=1
```

#### 4. Monitoring & Alerts

- [ ] Check error rate in Sentry (should be 0)
- [ ] Check response times in Datadog (P95 < 500ms)
- [ ] Check database connections (pool healthy)
- [ ] Check indexer status (not lagging)
- [ ] View real-time logs

#### 5. Rollback Plan

**If deployment fails**:

```bash
# Revert to previous version
git revert <commit-hash>
git push

# Rollback database migrations
pnpm run db:rollback

# Redeploy previous version
# (see deployment platform-specific docs)

# Notify team in #incidents channel
```

## Deployment Platforms

### AWS EC2 + RDS

```bash
# Launch EC2 instance
aws ec2 run-instances --image-id ami-xxxxx --instance-type t3.medium

# Create RDS database
aws rds create-db-instance \
  --db-instance-identifier dongle-prod \
  --engine postgres \
  --db-instance-class db.t3.medium \
  --allocated-storage 100

# Deploy application
# 1. SSH into EC2
# 2. Clone repository
# 3. npm install & npm run build
# 4. Configure systemd service
# 5. Start service
```

### AWS Lambda + RDS

```bash
# Build Lambda bundle
pnpm run build:lambda

# Deploy to Lambda
serverless deploy --stage production

# Configure API Gateway for REST endpoints
```

### Google Cloud Run

```bash
# Build and deploy
gcloud run deploy dongle-backend \
  --source . \
  --region us-central1 \
  --set-env-vars DATABASE_URL=... \
  --memory 512M \
  --cpu 1
```

### Docker Compose (Staging)

```bash
# Build image
docker build -t dongle-backend:latest .

# Run with docker-compose
docker-compose -f docker-compose.prod.yml up -d

# Check logs
docker-compose logs -f app
```

## Monitoring & Observability

### Key Metrics to Track

| Metric | Target | Alert Threshold |
|--------|--------|-----------------|
| Response Time (P95) | < 200ms | > 500ms |
| Error Rate | < 0.1% | > 1% |
| Database Latency | < 50ms | > 200ms |
| CPU Usage | 30-50% | > 80% |
| Memory Usage | 50-70% | > 85% |
| Disk Space | 30-50% used | > 80% used |
| Indexer Lag | < 1 min | > 5 min |
| Cache Hit Rate | > 80% | < 50% |

### Logging

**Log aggregation service** (ELK, Datadog, CloudWatch):

```
Application logs → Syslog/Stdout → Log aggregator → Dashboard
```

**Key logs to monitor**:
- Errors and exceptions
- Slow queries (> 1s)
- Authentication failures
- Admin actions (verification approvals)
- Indexer event processing
- Database migrations

### Health Checks

```bash
# Configure load balancer health checks
GET /health/ready HTTP/1.1

# Expected response (200):
{
  "database": "ok",
  "soroban_rpc": "ok",
  "indexer": "ok"
}

# If any component returns non-ok, mark server unhealthy
```

## Backup & Disaster Recovery

### Database Backups

```bash
# Daily automated backups (RDS handles this)
# Configure: automated backup retention = 30 days

# Manual backup before major deployments
aws rds create-db-snapshot \
  --db-instance-identifier dongle-prod \
  --db-snapshot-identifier dongle-prod-backup-20241106

# Store backups in separate region for disaster recovery
aws rds copy-db-snapshot \
  --source-db-snapshot-identifier dongle-prod-backup-20241106 \
  --target-db-snapshot-identifier dongle-prod-backup-dr \
  --destination-region us-west-2
```

### Disaster Recovery Runbook

**If database is corrupted**:

1. Stop all API servers
2. Create snapshot from latest backup
3. Restore to new database instance
4. Verify data integrity
5. Update DATABASE_URL in environment
6. Restart API servers
7. Monitor indexer for recovery

**If all API servers fail**:

1. Check CloudWatch logs for error patterns
2. Verify database is accessible
3. Check Soroban RPC connectivity
4. Review recent deployments for issues
5. If infrastructure issue: restore from backup deployment
6. If code issue: revert to previous version

## Security in Production

### SSL/TLS

```bash
# Obtain certificate (Let's Encrypt)
certbot certonly --standalone -d api.dongle.app

# Configure in reverse proxy (nginx/Apache)
server {
  listen 443 ssl http2;
  ssl_certificate /etc/letsencrypt/live/api.dongle.app/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/api.dongle.app/privkey.pem;
  
  location / {
    proxy_pass http://localhost:3001;
  }
}

# Enforce HTTPS
server {
  listen 80;
  return 301 https://$server_name$request_uri;
}
```

### Network Security

- [ ] Database only accessible from API servers (security group rules)
- [ ] API servers behind load balancer with rate limiting
- [ ] DDoS protection enabled (CloudFlare, AWS Shield)
- [ ] WAF rules configured (ModSecurity, AWS WAF)
- [ ] Secrets in Vault/Secrets Manager (not in code)

### Access Control

- [ ] SSH key-based authentication only (no passwords)
- [ ] VPN access to production database (no public IP)
- [ ] Audit logging for all admin access
- [ ] Principle of least privilege for IAM roles
- [ ] Regular security audits and penetration testing

## Performance Tuning

### Database Optimization

```sql
-- Analyze query performance
EXPLAIN ANALYZE SELECT * FROM projects WHERE category = 'defi';

-- Create covering indexes for common queries
CREATE INDEX idx_projects_listing ON projects(category, verified, rating DESC)
  WHERE is_active = TRUE;

-- Enable query parallelization
SET max_parallel_workers_per_gather = 4;

-- Increase work memory for sorting
SET work_mem = '256MB';
```

### Connection Pooling

```
Application → PgBouncer (pool: 50 connections) → PostgreSQL
```

### Caching Strategy

- Cache project listings: 5 minute TTL
- Cache individual projects: 10 minute TTL
- Cache ratings/aggregations: 5 minute TTL
- Invalidate on write operations

## Cost Management

### Resource Sizing

| Component | Sizing | Est. Cost/month |
|-----------|--------|-----------------|
| **API Servers** | 3 × t3.medium | $50 |
| **Database** | db.t3.large | $150 |
| **Cache** | redis-5gb | $50 |
| **Load Balancer** | 1 ALB | $20 |
| **CDN** | CloudFront | $20-100 |
| **Monitoring** | Sentry + Datadog | $100 |
| **DNS** | Route 53 | $5 |
| **Total** | | **~$400-500/month** |

### Cost Optimization

- Use reserved instances (1-year commit): 25% discount
- Use spot instances for non-critical workloads
- Right-size database (don't over-provision)
- Implement auto-scaling to handle traffic peaks
- Clean up old logs/backups

## Maintenance

### Regular Tasks

| Task | Frequency | Owner |
|------|-----------|-------|
| Review error logs | Daily | On-call engineer |
| Monitor performance metrics | Daily | On-call engineer |
| Database maintenance (VACUUM, ANALYZE) | Weekly | DBA |
| Security patches | As needed | DevOps |
| Dependency updates | Monthly | Engineering team |
| Database backups verification | Monthly | DBA |
| Disaster recovery drill | Quarterly | DevOps |
| Security audit | Quarterly | Security team |

### Upgrade Process

```bash
# Test upgrade in staging first
git checkout v1.1.0
pnpm install
pnpm run build
pnpm run test

# If staging passes, upgrade production
# 1. Create database backup
# 2. Run migrations (if any)
# 3. Deploy new version
# 4. Monitor error rate and performance
# 5. If issues: rollback to previous version
```

## Support & Escalation

### On-Call Rotation

- Primary: Engineering lead
- Secondary: Senior engineer
- Backup: DevOps engineer

### Escalation Path

1. **Alert fires** → Check Sentry/Datadog
2. **Error rate < 1%** → Monitor, may be transient
3. **Error rate 1-5%** → Investigate, check recent changes
4. **Error rate > 5%** → Page on-call engineer, initiate incident
5. **Database down** → Immediate escalation, page multiple people
6. **Data corruption** → Activate disaster recovery plan

### Communication

- Incident channel: #incidents
- Status page: https://status.dongle.app
- User notification: Email + in-app notification

---

**Note**: This guide documents the target deployment process. Implementation details will be added as development progresses.

**See** [README.md](../README.md) for quick start and local development instructions.
