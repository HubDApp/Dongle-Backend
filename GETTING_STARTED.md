# Dongle Backend — Getting Started Guide

Welcome to the Dongle Backend project! This guide will help you understand the project structure and get up to speed quickly.

## 📋 First Time Here? Start Here

### 1. Read the Main README (5 minutes)
Start with [README.md](README.md) to understand:
- What Dongle Backend does
- High-level architecture
- Quick start commands
- Key features

### 2. Check Project Status (2 minutes)
Review [BUILD_STATUS.md](BUILD_STATUS.md) to see:
- Current development phase
- What's complete vs. in progress
- Deployment readiness
- Next steps in priority order

### 3. Understand the Architecture (15 minutes)
Read [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) to learn:
- Layered architecture (routes → controllers → services → repositories)
- Data flows and system design
- Authentication and authorization
- Error handling patterns

### 4. Explore the Database Design (10 minutes)
Review [docs/DATABASE_SCHEMA.md](docs/DATABASE_SCHEMA.md) to see:
- Core tables and relationships
- Indexes for performance
- Query patterns
- Data integrity constraints

### 5. Review API Specification (10 minutes)
Check [docs/API.md](docs/API.md) for:
- All REST endpoints planned
- Request/response examples
- Error codes and handling
- Pagination and filtering

**Total onboarding time: ~45 minutes**

## 🗺️ Project Map

### Core Documentation Files

```
README.md                  ← Start here! Project overview & quick start
BUILD_STATUS.md            ← Current project status & deployment readiness
.env.example               ← Configuration template
CHANGELOG.md               ← Release history (will be updated during dev)
LICENSE                    ← MIT License
GETTING_STARTED.md         ← You are here!

docs/
├── ARCHITECTURE.md        ← System design, layered architecture, patterns
├── API.md                 ← REST endpoint reference with examples
├── DATABASE_SCHEMA.md     ← PostgreSQL tables, relationships, indexes
├── DEPLOYMENT.md          ← Production deployment guide & checklist
├── INDEXING.md            ← Soroban event indexing & sync
└── CONTRIBUTING.md        ← Development guidelines, PR process, testing
```

### How to Use Each Document

| Document | When to Read | Key Questions It Answers |
|----------|-------------|-------------------------|
| **README.md** | First day | What is this? How do I start? What's the feature set? |
| **BUILD_STATUS.md** | Before deploying | Is this production-ready? What's missing? |
| **ARCHITECTURE.md** | Before coding | How should I structure this? Where does X go? |
| **API.md** | When building endpoints | What should this endpoint return? What parameters? |
| **DATABASE_SCHEMA.md** | When adding features | What tables exist? How are they related? What indexes? |
| **DEPLOYMENT.md** | Before going live | How do I deploy? What's the checklist? Monitoring? |
| **INDEXING.md** | When working on sync | How do events work? How is data synced? Recovery? |
| **CONTRIBUTING.md** | When submitting code | How should I commit? What's the PR process? Code style? |

## 🎯 Common Tasks

### "I want to understand the system"
→ Read in this order:
1. README.md (overview)
2. ARCHITECTURE.md (design)
3. DATABASE_SCHEMA.md (data model)

### "I want to add a new API endpoint"
→ Follow this guide:
1. Read ARCHITECTURE.md (layers)
2. Check API.md (endpoint patterns)
3. Read CONTRIBUTING.md (feature template)
4. Follow the 6-step process in CONTRIBUTING.md

### "I want to set up development environment"
→ Use this checklist:
1. Clone repository
2. Copy `.env.example` to `.env.local`
3. Install Node.js 20.x
4. Run `pnpm install`
5. Set up PostgreSQL
6. Run `pnpm run db:migrate`
7. Run `pnpm run dev`

See README.md for detailed commands.

### "I want to understand the database"
→ Follow this path:
1. Read DATABASE_SCHEMA.md (overview)
2. Review the core tables section
3. Check the relationships diagram
4. Look at query patterns
5. Review indexing strategy

### "I want to deploy to production"
→ Use DEPLOYMENT.md:
1. Review deployment architecture
2. Check prerequisites
3. Follow the deployment checklist
4. Monitor using health endpoints
5. Have rollback plan ready

### "I want to understand event indexing"
→ Read INDEXING.md:
1. Event types and handlers
2. Indexer service flow
3. Error handling & recovery
4. Monitoring

## 🧠 Understanding the Codebase

### Mental Model

```
Frontend sends REST request
    ↓
Express Route Handler (src/api/*/routes.ts)
    ↓
Controller (src/api/*/controller.ts) — Orchestration
    ↓
Service (src/services/*.ts) — Business Logic
    ↓
Repository (src/repositories/*.ts) — Data Access
    ↓
PostgreSQL Database
```

**Rule**: Data flows DOWN, responses flow UP

### Where Code Goes

- **Routes** (`src/api/*/routes.ts`): Define HTTP endpoints
- **Controllers** (`src/api/*/controller.ts`): Format requests/responses
- **Services** (`src/services/*.ts`): Business logic and features
- **Repositories** (`src/repositories/*.ts`): Database queries
- **Models** (`src/models/*.ts`): TypeScript types
- **Middleware** (`src/middleware/*.ts`): Cross-cutting concerns
- **Utils** (`src/utils/*.ts`): Shared helper functions
- **Tests** (`tests/`): Mirror the source structure

### Adding a Feature Checklist

1. **Design** → Add to docs/API.md
2. **Route** → Create in src/api/*/routes.ts
3. **Controller** → Create in src/api/*/controller.ts
4. **Service** → Create in src/services/*.ts
5. **Repository** → Add methods in src/repositories/*.ts
6. **Database** → Create migration if needed
7. **Test** → Add tests in tests/
8. **Update Docs** → Update API.md with example

See CONTRIBUTING.md section "Adding a New Endpoint" for the detailed 6-step process.

## 🔑 Key Concepts

### Event-Driven Indexing

Soroban smart contracts emit events → Indexer polls events → Processes with handlers → Updates database

**Key insight**: Database is a cache of smart contract state, not the source of truth.

### Service + Repository Pattern

- **Services**: Business logic (what to do)
- **Repositories**: Data access (how to store/retrieve)

This separation makes testing easy:
- Mock repositories for service tests
- Mock services for controller tests

### Layered Architecture

Each layer has one responsibility:
- **Routes**: Parse HTTP requests
- **Controllers**: Coordinate services, format responses
- **Services**: Business rules
- **Repositories**: Database queries

Benefits: Easy to test, understand, and modify.

## 🛠️ Development Workflow

### For Backend Development

```bash
# 1. Set up environment
cp .env.example .env.local
# Edit .env.local

# 2. Install dependencies
pnpm install

# 3. Start database
# (PostgreSQL must be running)

# 4. Run migrations
pnpm run db:migrate

# 5. Start dev server
pnpm run dev

# Server runs at http://localhost:3001
```

### For Writing Code

```bash
# 1. Create feature branch
git checkout -b feature/your-feature

# 2. Write code following patterns in CONTRIBUTING.md
# 3. Add tests
# 4. Run linters
pnpm run lint && pnpm run typecheck

# 5. Run tests
pnpm run test

# 6. Commit with conventional commits
git commit -m "feat(projects): add search filtering"

# 7. Push and create PR
git push origin feature/your-feature
```

### Commit Message Format

```
<type>(<scope>): <subject>

<body>

<footer>
```

**Types**: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`, `perf`

**Examples**:
```
feat(reviews): add review aggregation service
fix(projects): fix filtering by category
docs(api): update project endpoint documentation
test(services): add tests for verification workflow
```

## 🐛 Debugging Tips

### Check Logs

```bash
# Terminal output
npm run dev

# Log files (if configured)
tail -f logs/app.log
```

### Inspect Database

```bash
# Connect to PostgreSQL
psql $DATABASE_URL

# List tables
\dt

# View project data
SELECT * FROM projects LIMIT 5;
```

### Test an Endpoint

```bash
# Using curl
curl http://localhost:3001/api/health

# Using httpie
http GET localhost:3001/api/projects limit==10

# Using VS Code REST Client
# Create file.http and use extension
```

## 📚 Learning Resources

### Documentation Links
- [Stellar Developer Docs](https://developers.stellar.org/)
- [Soroban Documentation](https://soroban.stellar.org/)
- [Node.js Best Practices](https://nodejs.dev/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [Express.js Guide](https://expressjs.com/)
- [TypeScript Handbook](https://www.typescriptlang.org/docs/)

### Books/Articles
- "Clean Architecture" by Robert C. Martin (layered architecture concepts)
- "The Pragmatic Programmer" (general best practices)
- "SQL Performance Explained" (for database optimization)

### Videos
- Soroban intro (if available)
- PostgreSQL query optimization (various sources)
- Node.js best practices (various sources)

## ❓ FAQ

**Q: Where should I add this new code?**
A: Check ARCHITECTURE.md section "Data Flow Diagrams" and CONTRIBUTING.md section "Adding a New Endpoint"

**Q: How do I test my changes?**
A: See CONTRIBUTING.md section "Testing" for test patterns

**Q: What's the coding style?**
A: See CONTRIBUTING.md section "Code Style & Linting"

**Q: How do I commit my changes?**
A: See CONTRIBUTING.md section "Commit Guidelines" for conventional commits

**Q: Is the system production-ready?**
A: Check BUILD_STATUS.md for current status. Currently in alpha development.

**Q: How do I deploy?**
A: Read DEPLOYMENT.md when ready. Currently not production-ready.

**Q: How are smart contract events processed?**
A: See INDEXING.md for the full event handling pipeline

**Q: What's the database schema?**
A: Check DATABASE_SCHEMA.md for tables, relationships, and indexes

## 🚀 Next Steps

### If You Want To...

**Contribute Code**
1. Read docs/CONTRIBUTING.md
2. Check the feature template
3. Follow the 6-step process for new endpoints

**Understand the System**
1. Read ARCHITECTURE.md thoroughly
2. Study DATABASE_SCHEMA.md
3. Review API.md
4. Check INDEXING.md

**Deploy the Project**
1. Review BUILD_STATUS.md (not ready yet)
2. Read DEPLOYMENT.md
3. Follow the deployment checklist

**Add a New Endpoint**
1. Document in API.md
2. Follow the 6-step process in CONTRIBUTING.md
3. Add tests
4. Update documentation

## 📞 Need Help?

1. **Check the docs** — Most questions are answered in the documentation
2. **Read CONTRIBUTING.md** — Development questions answered there
3. **Search the codebase** — Existing code shows patterns
4. **Open a GitHub issue** — For bugs or feature requests
5. **Start a discussion** — For questions about design

---

**You're all set! Ready to start?** Pick a task from [BUILD_STATUS.md](BUILD_STATUS.md) and refer back to these docs as needed.

Happy coding! 🚀
