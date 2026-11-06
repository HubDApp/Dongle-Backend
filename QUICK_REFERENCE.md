# Dongle Backend — Quick Reference Card

## 📖 Documentation at a Glance

### Which File Do I Read?

| I Want To... | Read This | Time |
|--------------|-----------|------|
| Get started | README.md | 5 min |
| Understand the system | ARCHITECTURE.md | 15 min |
| Check project status | BUILD_STATUS.md | 2 min |
| Learn database design | DATABASE_SCHEMA.md | 10 min |
| See API endpoints | API.md | 10 min |
| Deploy to production | DEPLOYMENT.md | 20 min |
| Understand event sync | INDEXING.md | 10 min |
| Write code | CONTRIBUTING.md | 10 min |
| **Get up to speed** | **GETTING_STARTED.md** | **45 min** |

## 🗂️ File Organization

```
Dongle-Backend/
├── README.md                    ← Start here
├── GETTING_STARTED.md          ← Then here
├── BUILD_STATUS.md             ← Check status
├── CHANGELOG.md                ← Version history
├── .env.example                ← Configuration
├── LICENSE                     ← MIT
├── .gitignore                  ← Git patterns
└── docs/
    ├── API.md                  ← Endpoints
    ├── ARCHITECTURE.md         ← Design
    ├── DATABASE_SCHEMA.md      ← Tables
    ├── DEPLOYMENT.md           ← Deploy
    ├── INDEXING.md             ← Events
    └── CONTRIBUTING.md         ← Dev guide
```

## 🎯 Quick Answers

### Architecture
- **Pattern**: Layered (Routes → Controllers → Services → Repositories)
- **Language**: Node.js/TypeScript/Express
- **Database**: PostgreSQL
- **Blockchain**: Stellar/Soroban

### Core Tables
- `projects` - Registered projects
- `reviews` - Community ratings
- `verification_requests` - Verification workflow
- `users` - User profiles
- `audit_logs` - Admin actions

### API Endpoints (Groups)
- **Projects**: GET, POST, PATCH (CRUD + search)
- **Reviews**: GET, POST, PATCH, DELETE
- **Verification**: POST request, GET status, admin approve/reject
- **Users**: GET profile, PATCH, GET projects/reviews
- **Admin**: Health checks, indexer status

### Key Events (Soroban)
1. ProjectRegistered → Create project
2. ReviewSubmitted → Aggregate rating
3. VerificationRequested → Track request
4. VerificationApproved → Mark verified
5. ProjectUpdated → Update metadata

## 💾 Database Reference

### Key Tables
```
projects(id, name, owner, verified, rating, review_count)
reviews(id, project_id, reviewer, rating, comment)
verification_requests(id, project_id, status)
users(address, display_name, reputation_score)
audit_logs(id, action, actor, subject_id, changes)
```

### Indexes
```
projects: (owner), (category), (verified), (rating DESC), GIN(name)
reviews: (project_id), (reviewer), (rating), (created_at DESC)
verification_requests: (status), (created_at DESC)
audit_logs: (actor), (action), (timestamp DESC)
```

## 🚀 Development

### Quick Start
```bash
# Setup
pnpm install
cp .env.example .env.local
pnpm run db:migrate

# Dev server
pnpm run dev

# Testing
pnpm run test
pnpm run test:watch

# Code quality
pnpm run lint && pnpm run typecheck
```

### Feature Template (6 Steps)
1. **Design**: Add to docs/API.md
2. **Route**: Create in src/api/*/routes.ts
3. **Controller**: Create in src/api/*/controller.ts
4. **Service**: Create in src/services/*.ts
5. **Repository**: Add in src/repositories/*.ts
6. **Test**: Add tests in tests/

### Commit Format
```
type(scope): subject

body (optional)

footer (optional)
```

**Types**: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`, `perf`

## 🔐 Security

### Key Points
- ✅ JWT authentication required for writes
- ✅ Parameterized queries (no SQL injection)
- ✅ Input validation with Zod
- ✅ CORS restricted to frontend domain
- ✅ Rate limiting per IP
- ✅ Audit logging for all admin actions

### Environment Variables
- `JWT_SECRET` - Min 32 chars (required)
- `DATABASE_URL` - PostgreSQL connection
- `SOROBAN_RPC_URL` - Contract interaction
- `ADMIN_ALLOWLIST` - Admin addresses (comma-separated)

## 📊 Monitoring

### Health Checks
```bash
GET /health           # Liveness (is server running?)
GET /health/ready     # Readiness (all deps working?)
GET /admin/health     # Detailed status
```

### Key Metrics
| Metric | Target | Alert |
|--------|--------|-------|
| Response Time (P95) | < 200ms | > 500ms |
| Error Rate | < 0.1% | > 1% |
| Indexer Lag | < 1 min | > 5 min |
| Database Latency | < 50ms | > 200ms |
| Cache Hit Rate | > 80% | < 50% |

## 🐛 Troubleshooting

| Problem | Check |
|---------|-------|
| Database error | Is PostgreSQL running? Check DATABASE_URL |
| Indexer not syncing | Is SOROBAN_RPC_URL accessible? Check logs |
| API returning 401 | JWT_SECRET set? Token valid? |
| Slow queries | Check indexes? Run EXPLAIN ANALYZE? |
| Build failing | pnpm install? typecheck pass? lint pass? |

## 🚢 Deployment

### Checklist
- [ ] Environment variables set
- [ ] Database migrations run
- [ ] Contract IDs configured
- [ ] JWT secret (32+ chars, unique)
- [ ] CORS domain configured
- [ ] Monitoring set up
- [ ] Backups configured
- [ ] SSL/TLS in place
- [ ] Admin addresses configured
- [ ] Health checks passing

### Deployment Platforms
- AWS EC2 + RDS
- Google Cloud Run
- AWS Lambda
- Docker/Kubernetes
- Any Node.js hosting

## 📚 Key Concepts

### Event-Driven Indexing
Smart contract emits → Indexer polls → Processes → Updates database

### Service Layer
Business logic isolated from data access for testability

### Layered Architecture
Each layer has one responsibility, easy to test and modify

### Source of Truth
Soroban is the source; database is optimized cache

### Aggregation
Database computes ratings, counts, statistics (Soroban can't)

## 🔗 Links & References

### Documentation
- [README.md](README.md) - Project overview
- [ARCHITECTURE.md](docs/ARCHITECTURE.md) - System design
- [API.md](docs/API.md) - Endpoints
- [DATABASE_SCHEMA.md](docs/DATABASE_SCHEMA.md) - Schema
- [DEPLOYMENT.md](docs/DEPLOYMENT.md) - Deploy
- [INDEXING.md](docs/INDEXING.md) - Events
- [CONTRIBUTING.md](docs/CONTRIBUTING.md) - Dev guide

### External Resources
- [Stellar Docs](https://developers.stellar.org/)
- [Soroban Docs](https://soroban.stellar.org/)
- [PostgreSQL Docs](https://www.postgresql.org/docs/)
- [Express.js](https://expressjs.com/)
- [TypeScript](https://www.typescriptlang.org/)

## ✅ Project Status

| Component | Status | Notes |
|-----------|--------|-------|
| Documentation | ✅ Complete | All docs ready |
| Architecture | ✅ Designed | System design finalized |
| Project Setup | 🟡 Ready | Not started |
| Core Services | 🟡 Ready | Can implement now |
| API Endpoints | 🟡 Ready | Fully specified |
| Database | 🟡 Ready | Schema designed |
| Indexing | 🟡 Ready | Events documented |
| Testing | 🟡 Ready | Guidelines provided |
| Deployment | ❌ Not Ready | In alpha stage |

**See [BUILD_STATUS.md](BUILD_STATUS.md) for full details**

## 🎓 Learning Paths

### 5-Minute Overview
1. README.md (5 min)

### 1-Hour Understanding
1. README.md (5 min)
2. ARCHITECTURE.md (15 min)
3. API.md (20 min)
4. DATABASE_SCHEMA.md (10 min)
5. CONTRIBUTING.md (10 min)

### Full Deep Dive (3 hours)
All documentation files in order

---

**Stuck?** Check the appropriate doc or search for keywords in the documentation.

**Ready to start?** Follow [GETTING_STARTED.md](GETTING_STARTED.md) for a guided onboarding.

**Want to contribute?** Read [CONTRIBUTING.md](docs/CONTRIBUTING.md) for the dev workflow.
