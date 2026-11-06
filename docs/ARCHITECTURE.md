# Dongle Backend — Architecture Overview

## System Overview

The Dongle Backend is a Node.js/TypeScript API server that bridges the Dongle Smart Contract on Stellar with the frontend application. It provides persistent data storage, real-time indexing, and aggregated analytics while maintaining the decentralized integrity of on-chain data.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Frontend (Web/Mobile)                        │
│                        (Dongle App)                              │
└─────────────────────────────────────────────────────────────────┘
                              ↕ (REST/WebSocket)
┌─────────────────────────────────────────────────────────────────┐
│                      Dongle Backend API                          │
│  (Node.js/Express/TypeScript on Port 3001)                      │
├─────────────────────────────────────────────────────────────────┤
│  Route Layer           │  Service Layer          │ Data Access   │
│  ────────────────────  │  ──────────────────────  │ ──────────    │
│  • Projects            │  • ProjectService       │ • Projects    │
│  • Reviews             │  • ReviewService        │ • Reviews     │
│  • Verification        │  • VerificationService  │ • Verification│
│  • Admin               │  • UserService          │ • Users       │
│  • Health              │  • IndexerService       │ • Audit Logs  │
│  • WebSocket           │  • CacheService         │               │
└─────────────────────────────────────────────────────────────────┘
                              ↕
┌─────────────────────────────────────────────────────────────────┐
│                   PostgreSQL Database                            │
│  (Persistent Storage, Indexing, Aggregation)                    │
└─────────────────────────────────────────────────────────────────┘
                              ↕ (Monitor Events)
┌─────────────────────────────────────────────────────────────────┐
│              Soroban Smart Contract on Stellar                   │
│  (Source of Truth: Projects, Reviews, Verification)             │
│                                                                  │
│  Events:                                                         │
│  • ProjectRegistered → Backend creates project                  │
│  • ReviewSubmitted → Backend aggregates rating                  │
│  • VerificationRequested → Backend tracks request               │
│  • VerificationApproved → Backend updates status                │
└─────────────────────────────────────────────────────────────────┘
```

## Layered Architecture

### 1. Route Layer (`src/api/`)

Defines Express routes and maps HTTP requests to controllers.

**Responsibilities**:
- Parse and validate incoming HTTP requests
- Call appropriate controller methods
- Return formatted JSON responses
- Error handling and status codes

**Files**:
- `projects/routes.ts` — Project endpoints (GET, POST, PATCH)
- `reviews/routes.ts` — Review endpoints (GET, POST, DELETE)
- `verification/routes.ts` — Verification request endpoints
- `admin/routes.ts` — Admin-only endpoints
- `health/routes.ts` — Health check endpoints

**Example Route**:
```typescript
// GET /api/projects
router.get('/', async (req, res) => {
  const { page, limit, search } = req.query;
  const projects = await projectController.list(page, limit, search);
  res.json(projects);
});
```

### 2. Controller Layer (`src/api/*/controller.ts`)

Business logic orchestration — combines services and formats responses.

**Responsibilities**:
- Orchestrate multiple services
- Format responses for client consumption
- Validate input schemas
- Handle controller-level error handling

**Example**:
```typescript
export class ProjectController {
  async list(page: number, limit: number, search?: string) {
    const projects = await this.projectService.list({
      page,
      limit,
      search,
    });
    
    return {
      data: projects.items,
      pagination: {
        page: projects.page,
        limit: projects.limit,
        total: projects.total,
      },
    };
  }
}
```

### 3. Service Layer (`src/services/`)

Core business logic — implements features, orchestrates data access.

**Responsibilities**:
- Business rules and validation
- Coordinate repositories and external services
- Aggregation and transformation
- Feature-specific logic

**Services**:

#### ProjectService
```typescript
class ProjectService {
  async register(data) → projectId         // Create new project
  async update(id, data) → Project         // Update metadata
  async get(id) → Project                  // Fetch with reviews
  async list(filter) → [Project]           // Paginated listing
  async search(query) → [Project]          // Full-text search
  async getStats(id) → ProjectStats        // Aggregated stats
}
```

#### ReviewService
```typescript
class ReviewService {
  async submit(projectId, review) → reviewId    // Create review
  async getByProject(projectId) → [Review]      // Get all reviews
  async aggregateRating(projectId) → Rating     // Calculate avg
  async report(reviewId, reason) → void         // Flag for moderation
}
```

#### VerificationService
```typescript
class VerificationService {
  async request(projectId, user) → requestId    // Create request
  async getStatus(projectId) → Status           // Current status
  async approve(projectId, admin) → void        // Admin action
  async reject(projectId, admin, reason) → void // Admin action
}
```

#### UserService
```typescript
class UserService {
  async getProfile(address) → UserProfile       // Get profile
  async updateProfile(address, data) → void     // Update info
  async getReputation(address) → Score          // Calc reputation
  async getSubmittedProjects(address) → [Project]
}
```

#### IndexerService
```typescript
class IndexerService {
  async start() → void                          // Start monitoring
  async sync(fromLedger) → SyncResult           // Sync events
  async handleEvent(event) → void               // Process event
}
```

### 4. Repository Layer (`src/repositories/`)

Data access abstraction — encapsulates database queries.

**Responsibilities**:
- Execute database queries
- Map database rows to domain objects
- Handle database transactions
- Query optimization and indexing

**Pattern**:
```typescript
class ProjectRepository {
  async create(data): Promise<Project>
  async findById(id): Promise<Project | null>
  async findAll(filters): Promise<Project[]>
  async update(id, data): Promise<void>
  async delete(id): Promise<void>
  async search(query): Promise<Project[]>
}
```

**Example Query**:
```typescript
async findByOwner(owner: string): Promise<Project[]> {
  const rows = await db.query(
    'SELECT * FROM projects WHERE owner = $1 ORDER BY created_at DESC',
    [owner]
  );
  return rows.map(row => this.mapToDomain(row));
}
```

### 5. Database Layer (`src/database/`)

PostgreSQL connection management, migrations, and schema.

**Responsibilities**:
- Connection pool management
- Migration execution
- Schema definition
- Query builder initialization

**Files**:
- `client.ts` — PostgreSQL pool setup
- `migrations/` — SQL migration files
- `seeds/` — Test data seeding

## Data Flow Diagrams

### 1. Project Discovery Flow

```
User searches projects in frontend
         ↓
GET /api/projects?search=stellar&limit=20
         ↓
ProjectController.list()
         ↓
ProjectService.search(query)
         ↓
ProjectRepository.search(query)
         ↓
SELECT * FROM projects WHERE name ILIKE '%stellar%' LIMIT 20
         ↓
Map DB rows to Project objects
         ↓
Return paginated results to client
         ↓
Frontend renders project cards
```

### 2. Review Aggregation Flow

```
Indexer detects ReviewSubmitted event on-chain
         ↓
IndexerService.handleEvent(event)
         ↓
ReviewService.submit(projectId, reviewData)
         ↓
ReviewRepository.create(review)
         ↓
INSERT INTO reviews VALUES(...)
         ↓
ReviewService.aggregateRating(projectId)
         ↓
SELECT AVG(rating), COUNT(*) FROM reviews WHERE project_id = ?
         ↓
UPDATE projects SET rating = ..., review_count = ... WHERE id = ?
         ↓
Cache invalidated
         ↓
Frontend sees updated rating when it refreshes
```

### 3. Verification Request Workflow

```
User requests project verification in frontend
         ↓
POST /api/verification/request { projectId, user }
         ↓
VerificationController.request()
         ↓
VerificationService.request(projectId, user)
         ↓
VerificationRepository.create(request)
         ↓
INSERT INTO verification_requests (status='PENDING')
         ↓
Emit notification event (potential WebSocket)
         ↓
Response: { requestId, status: 'PENDING', ... }
         ↓
Admin sees pending request in dashboard
         ↓
Admin approves: POST /api/admin/verification/:projectId/approve
         ↓
VerificationService.approve(projectId, adminAddress)
         ↓
UPDATE verification_requests SET status='APPROVED'
         ↓
UPDATE projects SET verified=true
         ↓
Trigger on-chain verification smart contract call
         ↓
Frontend receives update (refresh or WebSocket notification)
```

### 4. Event Indexing Flow

```
Soroban smart contract emits event
         ↓
Indexer polls Soroban RPC (every 6 seconds)
         ↓
GET https://soroban-testnet.stellar.org/events?startLedger=X
         ↓
IndexerService.sync() — process new events
         ↓
For each event:
  - ProjectRegistered → ProjectService.registerFromEvent()
  - ReviewSubmitted → ReviewService.submitFromEvent()
  - VerificationApproved → VerificationService.approveFromEvent()
         ↓
Update database tables
         ↓
Cache invalidation triggers
         ↓
WebSocket broadcasts updates to connected clients (future)
         ↓
Frontend receives real-time notification
```

## Database Schema (High Level)

### Core Tables

```
projects
├── id (PRIMARY KEY)
├── name
├── description
├── owner (Stellar address)
├── website_url
├── category
├── verified (bool)
├── rating (numeric: 1-5)
├── review_count (integer)
├── metadata_cid (IPFS CID)
├── created_at (timestamp)
├── updated_at (timestamp)
└── INDEX: (owner), (verified), (category)

reviews
├── id (PRIMARY KEY)
├── project_id (FOREIGN KEY → projects)
├── reviewer (Stellar address)
├── rating (integer: 1-5)
├── comment (text)
├── hidden (bool)
├── created_at (timestamp)
├── updated_at (timestamp)
└── INDEX: (project_id), (reviewer), (created_at DESC)

verification_requests
├── id (PRIMARY KEY)
├── project_id (FOREIGN KEY → projects, UNIQUE)
├── requested_by (Stellar address)
├── status (ENUM: PENDING, APPROVED, REJECTED)
├── rejected_reason (text)
├── created_at (timestamp)
├── approved_at (timestamp)
├── approved_by (Stellar address)
└── INDEX: (status), (created_at DESC)

users
├── address (PRIMARY KEY - Stellar address)
├── display_name (text)
├── bio (text)
├── avatar_url (text)
├── reputation_score (numeric)
├── project_count (integer)
├── review_count (integer)
├── created_at (timestamp)
├── updated_at (timestamp)
└── INDEX: (reputation_score DESC)

audit_logs
├── id (PRIMARY KEY)
├── action (text: 'verification_approve', 'project_delete', etc)
├── actor (Stellar address)
├── subject_id (text)
├── subject_type (text)
├── changes (jsonb)
├── timestamp (timestamp)
└── INDEX: (actor), (action), (timestamp DESC)
```

## Key Design Decisions

### 1. Service + Repository Pattern

**Why**: Separates business logic from data access, enabling:
- Easier testing (mock repositories)
- Testable business logic
- Database abstraction
- Reusable queries

### 2. Event-Driven Indexing

**Why**: Maintains sync between on-chain and off-chain:
- Soroban is source of truth
- Backend mirrors state for fast queries
- Enables real-time aggregations (ratings, counts)
- Audit trail of all changes

### 3. PostgreSQL for Off-Chain Data

**Why**: Enables features Soroban can't:
- Full-text search
- Aggregation (avg ratings, counts)
- Complex queries (trending, filtering)
- Audit logging
- Multi-index queries (owner + category + verified)

### 4. Cache Layer

**Why**: Improves performance:
- Cache project listings (5 min TTL)
- Cache individual projects (10 min TTL)
- Cache ratings/reviews (5 min TTL)
- Invalidate on write operations

## Authentication & Authorization

```
User sends request
    ↓
Extract JWT from Authorization header
    ↓
Verify JWT signature (JWT_SECRET)
    ↓
Extract Stellar address from token payload
    ↓
Check address against ADMIN_ALLOWLIST (for admin endpoints)
    ↓
Grant/deny access
    ↓
Attach address to request context
    ↓
Services use address for ownership checks (e.g., only owner can update project)
```

**Token Payload**:
```json
{
  "address": "GXXXXX",
  "iat": 1234567890,
  "exp": 1234654290,
  "iss": "dongle-backend"
}
```

## Error Handling Strategy

```
Application Error (Service Layer)
    ↓
Throw custom AppError with:
  - code (string: 'INVALID_PROJECT', 'UNAUTHORIZED', etc)
  - status (HTTP status: 400, 401, 404, 500)
  - message (user-friendly description)
    ↓
Express Error Middleware catches
    ↓
Log error to console/Sentry
    ↓
Return formatted JSON response:
  {
    "error": {
      "code": "INVALID_PROJECT",
      "message": "Project not found",
      "status": 404
    }
  }
    ↓
Frontend displays user-friendly error message
```

## Monitoring & Observability

### Logging Structure

**All logs in JSON format** for easy parsing:
```json
{
  "timestamp": "2024-11-06T10:30:00Z",
  "level": "info",
  "service": "ProjectService",
  "action": "registerProject",
  "projectId": "proj_123",
  "duration": 245,
  "userId": "GXXXXX",
  "message": "Project registered successfully"
}
```

### Key Metrics

- **API Response Times**: Per-endpoint latency
- **Database Query Times**: Slow query detection
- **Indexer Lag**: Time behind latest ledger
- **Cache Hit Rate**: Cache effectiveness
- **Error Rate**: Failed requests percentage
- **Active Users**: Concurrent connections

### Health Checks

```bash
# Liveness — server is running
GET /api/health/live
Response: 200 { "status": "ok" }

# Readiness — all dependencies available
GET /api/health/ready
Response: 200 {
  "database": "ok",
  "soroban_rpc": "ok",
  "indexer": "ok"
}

# Detailed status
GET /api/admin/health
Response: {
  "uptime": 3600000,
  "database": { "connected": true, "latency": 12 },
  "indexer": { "running": true, "lag": 2 }
}
```

## Scalability Considerations

### Horizontal Scaling

- **Stateless servers**: No session state in memory, all in DB
- **Connection pooling**: Share PostgreSQL connections across instances
- **Load balancing**: Route requests across API servers
- **Caching layer**: Redis for distributed cache

### Vertical Scaling

- **Database indexing**: Strategic indexes on query columns
- **Query optimization**: Pagination, select specific fields
- **Connection pool tuning**: Adjust min/max connections
- **Async processing**: Offload heavy operations to workers

### Future Optimization

- Add Redis for caching
- Implement request queuing for heavy operations
- Database read replicas for analytics queries
- Event streaming (Kafka) for real-time indexing

## Security Architecture

### Input Validation

```
HTTP Request
    ↓
Route-level validation (basic checks)
    ↓
Controller-level validation (Zod schemas)
    ↓
Service-level validation (business rules)
    ↓
Reject if invalid, return 400 Bad Request
```

### Data Protection

- Database credentials in environment variables only
- JWT secrets minimum 32 characters
- CORS restricted to frontend domain
- Rate limiting per IP
- SQL injection prevention via parameterized queries
- XSS prevention via JSON responses only

## Deployment Architecture

### Development

```
Local Machine
├── Node.js server (npm run dev)
├── PostgreSQL database (localhost:5432)
└── Soroban RPC (testnet endpoint)
```

### Production

```
Cloud Provider (AWS/GCP/Azure)
├── API servers (load balanced)
├── PostgreSQL RDS (managed, replicated, backed up)
├── Redis cache (optional)
├── Monitoring (Sentry, Datadog)
└── SSL/TLS termination (CloudFront/CDN)
```

See [docs/DEPLOYMENT.md](DEPLOYMENT.md) for detailed deployment guide.

---

**Next**: Review [docs/DATABASE_SCHEMA.md](DATABASE_SCHEMA.md) for the complete schema design.
