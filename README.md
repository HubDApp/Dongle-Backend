# Dongle Backend

**Dongle Backend** is the server-side API and data persistence layer for the Dongle ecosystem. It provides persistent project registries, review aggregation, verification request management, and indexing services for the Dongle smart contracts on Stellar.

## 🎯 Overview

Dongle Backend complements the [Dongle Smart Contract](https://github.com/dongle-xyz/dongle-smartcontract) by:

- **Persistent Data Storage**: Stores project metadata, reviews, verification requests, and user profiles
- **Indexing Service**: Continuously monitors Soroban events and syncs on-chain state with off-chain storage
- **API Gateway**: Provides REST endpoints for the frontend to query projects, reviews, and verification status
- **Aggregation & Analytics**: Computes project ratings, reputation scores, and ecosystem metrics
- **Admin Dashboard**: Backend for moderation, verification request management, and system configuration
- **Real-Time Updates**: WebSocket support for live notifications on verification approvals, new reviews, and project updates

## ⚠️ Build Status

**Current Status**: 🟡 **IN DEVELOPMENT** — See [BUILD_STATUS.md](BUILD_STATUS.md) for current state and deployment readiness.

## 🏗️ Architecture

```
Frontend (Dongle)
    ↓ (REST/WebSocket)
Dongle Backend API
    ├── Project Service (CRUD, search, indexing)
    ├── Review Service (aggregation, ratings)
    ├── Verification Service (request lifecycle)
    ├── User Service (profiles, reputation)
    └── Indexing Service (Soroban event sync)
    ↓
Database (PostgreSQL)
    ├── projects
    ├── reviews
    ├── verification_requests
    ├── users
    └── audit_logs

Smart Contract (Stellar)
    ↑ (event emission)
    Soroban Events
```

## 📋 Key Features

- **Project Management**: Create, read, update, list projects with full-text search and filtering
- **Review System**: Submit, aggregate, and display community reviews with 5-star ratings
- **Verification Workflow**: Manage project verification requests with status tracking and admin actions
- **User Profiles**: Track user reputation, submitted projects, and review history
- **Indexing**: Automatically sync smart contract events to database for real-time data availability
- **Analytics**: Project statistics, trending projects, category breakdowns, reputation metrics
- **Admin APIs**: Verification request management, project moderation, system health monitoring
- **WebSocket Support**: Real-time notifications for verification status changes and new community activity
- **Audit Logging**: Track all administrative actions and data modifications for compliance

## 🚀 Quick Start

### Prerequisites

- **Node.js**: 20.x LTS (pinned in `.nvmrc`)
- **PostgreSQL**: 14+ (local or managed service)
- **Docker** (optional): For containerized deployment
- **pnpm**: 10.x recommended (or npm 10.x)

### Installation

```bash
# Clone the repository
git clone https://github.com/dongle-xyz/dongle-backend.git
cd dongle-backend

# Install dependencies
pnpm install

# Set up environment variables
cp .env.example .env.local

# Run database migrations
pnpm run db:migrate

# Start development server
pnpm run dev
```

### Environment Setup

Create `.env.local` with:

```env
# Database
DATABASE_URL=postgresql://user:password@localhost:5432/dongle_dev

# Server
NODE_ENV=development
PORT=3001
LOG_LEVEL=debug

# Stellar/Soroban
STELLAR_NETWORK_PASSPHRASE=Test SDF Network ; September 2015
SOROBAN_RPC_URL=https://soroban-testnet.stellar.org

# Smart Contract IDs
PROJECT_REGISTRY_CONTRACT=C...
REVIEW_REGISTRY_CONTRACT=C...
VERIFICATION_REGISTRY_CONTRACT=C...

# Indexing
INDEXING_ENABLED=true
INDEXING_START_LEDGER=1000000

# Admin
ADMIN_ALLOWLIST=GXXXXX,GYYYYY

# JWT
JWT_SECRET=your-secret-key-min-32-chars

# Optional: Analytics & Monitoring
SENTRY_DSN=
DATADOG_API_KEY=
```

### Running Locally

```bash
# Install dependencies
pnpm install

# Run migrations
pnpm run db:migrate

# Start development server (with auto-reload)
pnpm run dev

# Run tests
pnpm run test

# Run linter & type check
pnpm run lint && pnpm run typecheck
```

**API available at**: `http://localhost:3001`

## 📁 Project Structure

```
src/
├── api/                          # Express route handlers
│   ├── projects/
│   │   ├── routes.ts            # GET /projects, POST /projects, etc.
│   │   └── controller.ts        # Business logic for project endpoints
│   ├── reviews/
│   ├── verification/
│   ├── admin/
│   └── health/
│
├── services/                     # Core business logic
│   ├── project.service.ts       # Project CRUD, search, indexing
│   ├── review.service.ts        # Review aggregation, ratings
│   ├── verification.service.ts  # Verification request lifecycle
│   ├── user.service.ts          # User profiles, reputation
│   ├── soroban.service.ts       # Smart contract interaction
│   └── indexer.service.ts       # Event monitoring & sync
│
├── repositories/                # Data access layer
│   ├── project.repository.ts
│   ├── review.repository.ts
│   ├── verification.repository.ts
│   └── user.repository.ts
│
├── models/                       # Database models & types
│   ├── project.ts
│   ├── review.ts
│   ├── verification.ts
│   └── user.ts
│
├── middleware/                   # Express middleware
│   ├── auth.ts                  # JWT verification
│   ├── errorHandler.ts          # Error handling
│   ├── requestLogger.ts         # Request logging
│   └── rateLimit.ts             # Rate limiting
│
├── database/                     # Database configuration
│   ├── client.ts                # Database pool setup
│   ├── migrations/              # SQL migration files
│   └── seeds/                   # Test data seeding
│
├── events/                       # Soroban event types
│   ├── types.ts                 # Event interfaces
│   └── handlers/                # Event processing
│
├── utils/                        # Utilities
│   ├── logger.ts                # Logging utility
│   ├── validators.ts            # Input validation
│   ├── stellar.ts               # Stellar address helpers
│   └── cache.ts                 # Caching utilities
│
├── config/                       # Configuration
│   ├── env.ts                   # Environment validation (Zod)
│   └── constants.ts             # App constants
│
├── app.ts                        # Express app setup
└── server.ts                     # Server entry point

tests/
├── unit/                         # Unit tests
├── integration/                  # Integration tests
└── fixtures/                     # Test data

docs/
├── API.md                        # API endpoint documentation
├── ARCHITECTURE.md               # System design & data flows
├── DEPLOYMENT.md                 # Deployment guide
├── INDEXING.md                   # Event indexing details
├── DATABASE_SCHEMA.md            # Database structure
└── VERIFICATION_WORKFLOW.md      # Verification process

.env.example                      # Environment template
package.json                      # Dependencies & scripts
tsconfig.json                     # TypeScript configuration
```

## 🔌 API Endpoints

### Projects

```bash
# List all projects (paginated, searchable, filterable)
GET /api/projects?page=1&limit=20&search=stellar&category=defi&verified=true

# Get single project with aggregated reviews
GET /api/projects/:id

# Create new project (requires wallet signature)
POST /api/projects
Body: { name, description, website, category, owner, metadata_cid }

# Update project (owner only)
PATCH /api/projects/:id
Body: { name, description, website, metadata_cid }

# Get project statistics
GET /api/projects/:id/stats

# List projects by owner
GET /api/projects?owner=GXXXXX

# Search projects with full-text search
GET /api/projects/search?q=stellar+payment
```

### Reviews

```bash
# List reviews for a project
GET /api/projects/:projectId/reviews?page=1&limit=20

# Get single review
GET /api/reviews/:reviewId

# Submit review (requires wallet signature)
POST /api/projects/:projectId/reviews
Body: { rating, comment, reviewer }

# Update review (reviewer only)
PATCH /api/reviews/:reviewId
Body: { rating, comment }

# Delete review (reviewer only)
DELETE /api/reviews/:reviewId

# Get user's reviews
GET /api/reviews?reviewer=GXXXXX

# Get trending reviews
GET /api/reviews/trending?limit=10
```

### Verification

```bash
# Submit verification request
POST /api/verification/request
Body: { projectId, submittedBy }

# Get verification status
GET /api/verification/projects/:projectId/status

# List verification requests (admin only)
GET /api/admin/verification/requests?status=pending&limit=50

# Approve verification (admin only)
POST /api/admin/verification/:projectId/approve
Body: { approvedBy, notes }

# Reject verification (admin only)
POST /api/admin/verification/:projectId/reject
Body: { rejectedBy, reason }

# Get verification request history
GET /api/verification/projects/:projectId/history
```

### Users

```bash
# Get user profile
GET /api/users/:address

# Update user profile
PATCH /api/users/:address
Body: { display_name, bio, avatar_url }

# Get user's submitted projects
GET /api/users/:address/projects

# Get user's review history
GET /api/users/:address/reviews

# Get user reputation score
GET /api/users/:address/reputation
```

### Admin

```bash
# System health check
GET /api/admin/health

# Indexer status
GET /api/admin/indexer/status

# Database statistics
GET /api/admin/stats

# Audit logs
GET /api/admin/audit-logs?action=verification_approve&limit=50

# Re-sync contract state
POST /api/admin/indexer/resync?startLedger=1000000
```

For complete API documentation, see [docs/API.md](docs/API.md).

## 🔄 Indexing & Event Sync

The indexing service continuously monitors Soroban events and syncs smart contract state:

```bash
# Start indexer service
pnpm run indexer:start

# Check indexer status
curl http://localhost:3001/api/admin/indexer/status

# Manual sync from specific ledger
curl -X POST http://localhost:3001/api/admin/indexer/resync?startLedger=1234567
```

**Monitored Events**:
- `ProjectRegistered` → Create project record
- `ProjectUpdated` → Update project metadata
- `ReviewSubmitted` → Create review, recalculate ratings
- `VerificationRequested` → Create verification request
- `VerificationApproved` → Update project verification status
- `VerificationRejected` → Update request status, notify user

See [docs/INDEXING.md](docs/INDEXING.md) for detailed event handling logic.

## 🗄️ Database Schema

Core tables:

| Table | Purpose |
|-------|---------|
| `projects` | Project metadata, owner, verification status, ratings |
| `reviews` | Community reviews with ratings, timestamps, author |
| `verification_requests` | Verification request lifecycle, status tracking |
| `users` | User profiles, reputation scores, activity |
| `audit_logs` | Admin actions, moderation decisions, system events |
| `project_dependencies` | Project dependency links and circular-reference protection |
| `featured_projects` | Admin-curated featured project lists |

See [docs/DATABASE_SCHEMA.md](docs/DATABASE_SCHEMA.md) for full schema with relationships and indexes.

## 🧪 Testing

```bash
# Run all tests
pnpm run test

# Run tests in watch mode
pnpm run test:watch

# Run specific test file
pnpm run test -- services/project.service.test.ts

# Generate coverage report
pnpm run test:coverage
```

## 🚀 Deployment

### Local Development

```bash
npm run dev
```

### Docker Deployment

```bash
# Build image
docker build -t dongle-backend .

# Run container
docker run -p 3001:3001 \
  -e DATABASE_URL=postgresql://... \
  -e SOROBAN_RPC_URL=https://soroban-testnet.stellar.org \
  dongle-backend
```

### Production Deployment

See [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md) for:
- Environment configuration for production
- Database setup and migrations
- SSL/TLS setup
- Load balancing and scaling
- Monitoring and alerting
- Backup and disaster recovery

### Deployment Checklist

- [ ] Environment variables set and validated
- [ ] Database migrations run successfully
- [ ] Contract IDs configured correctly
- [ ] Indexer can connect to Soroban RPC
- [ ] JWT secret configured (min 32 chars)
- [ ] Rate limiting configured
- [ ] Error tracking (Sentry) configured
- [ ] CORS properly configured for frontend
- [ ] Admin allowlist populated
- [ ] SSL certificates in place
- [ ] Health checks passing

## 📚 Documentation

| Doc | Purpose |
|-----|---------|
| [API.md](docs/API.md) | Complete API endpoint reference with examples |
| [ARCHITECTURE.md](docs/ARCHITECTURE.md) | System design, data flows, indexing strategy |
| [DATABASE_SCHEMA.md](docs/DATABASE_SCHEMA.md) | Database structure, relationships, indexes |
| [INDEXING.md](docs/INDEXING.md) | Event monitoring, contract sync, recovery |
| [VERIFICATION_WORKFLOW.md](docs/VERIFICATION_WORKFLOW.md) | Verification request lifecycle, admin actions |
| [DEPLOYMENT.md](docs/DEPLOYMENT.md) | Production deployment guide and checklist |
| [CONTRIBUTING.md](docs/CONTRIBUTING.md) | Development guidelines and PR process |
| [CHANGELOG.md](CHANGELOG.md) | Release history and breaking changes |

## 🔐 Security

- **JWT Authentication**: All write operations require signed requests
- **Input Validation**: All endpoints validate input using Zod schemas
- **Rate Limiting**: Per-IP rate limits prevent abuse
- **SQL Injection Prevention**: Parameterized queries via query builder
- **CORS**: Configured to accept requests from frontend domain only
- **Audit Logging**: All admin actions logged for compliance
- **Secrets Management**: Environment variables for sensitive config

See [docs/SECURITY.md](docs/SECURITY.md) for detailed security guidelines.

## 🛠️ Development

### Code Style & Linting

```bash
# Check code style (ESLint)
pnpm run lint

# Fix style issues automatically
pnpm run lint:fix

# Type checking
pnpm run typecheck
```

### Database Migrations

```bash
# Create new migration
pnpm run db:create-migration AddProjectCategory

# Run pending migrations
pnpm run db:migrate

# Rollback last migration
pnpm run db:rollback

# Reset database (development only)
pnpm run db:reset
```

### Adding a New Endpoint

1. Create route handler in `src/api/[feature]/routes.ts`
2. Create controller in `src/api/[feature]/controller.ts`
3. Create/update service in `src/services/[feature].service.ts`
4. Add repository methods in `src/repositories/[feature].repository.ts`
5. Add tests in `tests/integration/[feature].test.ts`
6. Document in [docs/API.md](docs/API.md)

## 🤝 Contributing

Contributions welcome! Please follow:

1. [CONTRIBUTING.md](docs/CONTRIBUTING.md) for guidelines
2. Conventional commit format (`feat:`, `fix:`, `docs:`, etc.)
3. Add tests for new features
4. Run `pnpm run lint && pnpm run typecheck && pnpm run test` before committing
5. Update documentation for API changes

## 📊 Monitoring & Observability

- **Logging**: Structured JSON logs via Winston
- **Error Tracking**: Sentry integration for error reporting
- **Metrics**: Datadog or Prometheus-compatible metrics
- **Health Checks**: Liveness and readiness endpoints
- **Request Tracing**: Distributed tracing support

See [docs/MONITORING.md](docs/MONITORING.md) for observability setup.

## 🐛 Troubleshooting

| Issue | Solution |
|-------|----------|
| Database connection error | Verify DATABASE_URL and PostgreSQL is running |
| Indexer not syncing | Check SOROBAN_RPC_URL is accessible; check logs for rate limit errors |
| API returning 401 | Ensure JWT_SECRET is set and matches frontend |
| Reviews not aggregating | Check indexer status; verify ReviewSubmitted events are being processed |
| Verification requests stuck | Check admin availability; verify admin addresses are in ADMIN_ALLOWLIST |

See [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) for detailed troubleshooting.

## 📞 Support

- **Issues**: Open an issue on GitHub
- **Discussions**: GitHub Discussions for Q&A
- **Stellar Docs**: [developers.stellar.org](https://developers.stellar.org/)
- **Soroban Docs**: [soroban.stellar.org](https://soroban.stellar.org/)

## 📄 License

MIT License — See [LICENSE](LICENSE) for details.

## 🙌 Acknowledgments

- Built on [Stellar](https://stellar.org/) and [Soroban](https://soroban.stellar.org/)
- Inspired by decentralized project registries
- Community-driven verification model

---

**Ready to build the future of Stellar discovery?** Start with the [Quick Start](#-quick-start) and dive into [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) to understand the system design.
