# Dongle Backend — Database Schema

## Overview

The database uses PostgreSQL to store persistent state that complements the Soroban smart contracts. The schema is designed for fast queries, efficient indexing, and support for advanced features like full-text search and aggregation.

## Core Tables

### projects

Stores project metadata registered on-chain.

```sql
CREATE TABLE projects (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  
  -- On-chain identity
  contract_id TEXT NOT NULL UNIQUE,          -- Soroban contract identifier
  owner TEXT NOT NULL,                        -- Stellar address (account owner)
  
  -- Project metadata
  name VARCHAR(255) NOT NULL,
  description TEXT,
  website_url VARCHAR(2048),
  category VARCHAR(50),                       -- defi, nft, payment, tools, other
  
  -- Verification and trust
  verified BOOLEAN DEFAULT FALSE,
  verification_status VARCHAR(20),            -- PENDING, APPROVED, REJECTED
  verified_at TIMESTAMP,
  verification_request_id UUID,               -- FK to verification_requests
  
  -- Aggregated data
  rating NUMERIC(3, 2),                       -- 1.0 to 5.0
  review_count INTEGER DEFAULT 0,
  last_reviewed_at TIMESTAMP,
  
  -- Off-chain metadata
  metadata_cid VARCHAR(255),                  -- IPFS CID for extended metadata
  
  -- Timestamps
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
  last_synced_at TIMESTAMP,                   -- Last event from smart contract
  
  -- Soft delete
  deleted_at TIMESTAMP,
  is_active BOOLEAN DEFAULT TRUE
);

-- Indexes for common queries
CREATE INDEX idx_projects_owner ON projects(owner);
CREATE INDEX idx_projects_category ON projects(category);
CREATE INDEX idx_projects_verified ON projects(verified);
CREATE INDEX idx_projects_rating ON projects(rating DESC);
CREATE INDEX idx_projects_created_at ON projects(created_at DESC);
CREATE INDEX idx_projects_name_gin ON projects USING GIN(to_tsvector('english', name));
CREATE INDEX idx_projects_is_active ON projects(is_active);
```

**Key Fields**:
- `contract_id`: Links to on-chain contract
- `owner`: Stellar address that registered project
- `verified`: Whether admin approved project
- `rating`: Aggregated from reviews table
- `metadata_cid`: Points to IPFS for extended data

### reviews

Stores community reviews and ratings.

```sql
CREATE TABLE reviews (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  
  -- Relationships
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  
  -- Review data
  reviewer TEXT NOT NULL,                     -- Stellar address
  rating INTEGER NOT NULL CHECK (rating >= 1 AND rating <= 5),
  comment TEXT NOT NULL,
  
  -- Moderation
  hidden BOOLEAN DEFAULT FALSE,               -- Admin can hide inappropriate reviews
  report_count INTEGER DEFAULT 0,
  flagged_at TIMESTAMP,
  
  -- Off-chain metadata
  metadata_cid VARCHAR(255),                  -- IPFS CID for extended review data
  
  -- Timestamps
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
  
  -- Uniqueness: one review per user per project
  UNIQUE(project_id, reviewer)
);

-- Indexes for common queries
CREATE INDEX idx_reviews_project_id ON reviews(project_id);
CREATE INDEX idx_reviews_reviewer ON reviews(reviewer);
CREATE INDEX idx_reviews_rating ON reviews(rating);
CREATE INDEX idx_reviews_created_at ON reviews(created_at DESC);
CREATE INDEX idx_reviews_hidden ON reviews(hidden);
```

**Key Features**:
- **UNIQUE constraint**: One review per user per project (prevents duplicates)
- **ON DELETE CASCADE**: Deleting project deletes its reviews
- `hidden`: Admin moderation (can hide spam/inappropriate reviews)
- `metadata_cid`: Extended review data on IPFS

### verification_requests

Tracks project verification request lifecycle.

```sql
CREATE TABLE verification_requests (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  
  -- Project being verified
  project_id UUID NOT NULL UNIQUE REFERENCES projects(id) ON DELETE CASCADE,
  
  -- Request tracking
  requested_by TEXT NOT NULL,                 -- Stellar address of requester
  status VARCHAR(20) NOT NULL DEFAULT 'PENDING',  -- PENDING, APPROVED, REJECTED
  
  -- Approval data (set when status changes)
  approved_by TEXT,                           -- Admin address
  approved_at TIMESTAMP,
  
  -- Rejection data
  rejected_by TEXT,                           -- Admin address
  rejected_at TIMESTAMP,
  rejection_reason TEXT,
  
  -- Renewal tracking
  renewal_requested_at TIMESTAMP,
  renewal_approved_at TIMESTAMP,
  
  -- Timestamps
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
  
  -- Audit
  contract_synced_at TIMESTAMP                -- When last synced to on-chain
);

-- Indexes
CREATE INDEX idx_verification_requests_project_id ON verification_requests(project_id);
CREATE INDEX idx_verification_requests_status ON verification_requests(status);
CREATE INDEX idx_verification_requests_requested_by ON verification_requests(requested_by);
CREATE INDEX idx_verification_requests_created_at ON verification_requests(created_at DESC);
CREATE INDEX idx_verification_requests_approval_status ON verification_requests(status, created_at DESC);
```

**Key Features**:
- `UNIQUE(project_id)`: One verification per project
- Status workflow: PENDING → APPROVED or REJECTED
- Tracks both approval and rejection with reasons
- Supports renewal flow

### users

User profiles and reputation tracking.

```sql
CREATE TABLE users (
  address TEXT PRIMARY KEY,                   -- Stellar address (unique identifier)
  
  -- Profile
  display_name VARCHAR(255),
  bio TEXT,
  avatar_url VARCHAR(2048),
  
  -- Reputation
  reputation_score NUMERIC(10, 2) DEFAULT 0, -- Calculated from activities
  
  -- Activity counts
  project_count INTEGER DEFAULT 0,            -- Projects user has registered
  review_count INTEGER DEFAULT 0,             -- Reviews user has submitted
  verification_approvals_given INTEGER DEFAULT 0,  -- (admin only)
  
  -- Verification
  is_admin BOOLEAN DEFAULT FALSE,
  verified_at TIMESTAMP,
  
  -- Timestamps
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
  last_activity_at TIMESTAMP
);

-- Indexes
CREATE INDEX idx_users_reputation_score ON users(reputation_score DESC);
CREATE INDEX idx_users_is_admin ON users(is_admin);
CREATE INDEX idx_users_created_at ON users(created_at DESC);
```

**Key Fields**:
- `address`: Stellar address (PK)
- `reputation_score`: Calculated metric based on reviews, projects, verification
- Activity counts used for profile display
- `is_admin`: Tracks admin status separately

### audit_logs

Records all administrative actions for compliance.

```sql
CREATE TABLE audit_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  
  -- Action details
  action TEXT NOT NULL,                       -- e.g., 'verification_approve', 'project_delete'
  actor TEXT NOT NULL,                        -- Stellar address who performed action
  actor_role VARCHAR(50),                     -- 'admin', 'user', 'system'
  
  -- What was affected
  subject_type VARCHAR(50),                   -- 'project', 'review', 'verification_request'
  subject_id UUID,
  
  -- Change tracking
  changes JSONB,                              -- What changed: { before: {...}, after: {...} }
  
  -- Additional context
  ip_address INET,
  user_agent TEXT,
  
  -- Timestamps
  timestamp TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_audit_logs_actor ON audit_logs(actor);
CREATE INDEX idx_audit_logs_action ON audit_logs(action);
CREATE INDEX idx_audit_logs_timestamp ON audit_logs(timestamp DESC);
CREATE INDEX idx_audit_logs_subject ON audit_logs(subject_type, subject_id);
```

**Key Features**:
- **JSONB for changes**: Flexible tracking of what changed
- Actor and role tracking for security
- IP address and user agent for debugging

### project_dependencies

Tracks relationships between projects.

```sql
CREATE TABLE project_dependencies (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  
  -- Relationship
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  depends_on_project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  
  -- Metadata
  dependency_type VARCHAR(50),                -- 'runtime', 'development', 'peer'
  description TEXT,
  
  -- Timestamps
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  
  -- Prevent duplicates
  UNIQUE(project_id, depends_on_project_id),
  
  -- Prevent self-reference
  CONSTRAINT no_self_dependency CHECK (project_id != depends_on_project_id)
);

-- Indexes
CREATE INDEX idx_project_dependencies_project_id ON project_dependencies(project_id);
CREATE INDEX idx_project_dependencies_depends_on ON project_dependencies(depends_on_project_id);
```

**Key Features**:
- Tracks project dependencies
- Prevents self-references and duplicate dependencies
- Used for circular-reference detection

### featured_projects

Admin-curated list of featured projects.

```sql
CREATE TABLE featured_projects (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  
  -- Featured item
  project_id UUID NOT NULL UNIQUE REFERENCES projects(id) ON DELETE CASCADE,
  
  -- Curation details
  featured_by TEXT NOT NULL,                  -- Admin address
  feature_reason TEXT,
  
  -- Ordering
  display_order INTEGER NOT NULL DEFAULT 0,
  
  -- Duration
  featured_at TIMESTAMP NOT NULL DEFAULT NOW(),
  featured_until TIMESTAMP,                   -- Optional: when to unfeature
  
  -- Timestamps
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_featured_projects_project_id ON featured_projects(project_id);
CREATE INDEX idx_featured_projects_display_order ON featured_projects(display_order ASC);
CREATE INDEX idx_featured_projects_featured_at ON featured_projects(featured_at DESC);
```

## Relationships

### Entity Relationship Diagram

```
┌─────────────┐
│   users     │
└──────┬──────┘
       │
       ├─ owns projects (user.address = projects.owner)
       ├─ submits reviews (user.address = reviews.reviewer)
       ├─ requests verification (user.address = verification_requests.requested_by)
       └─ approves verification (user.address = verification_requests.approved_by)

┌─────────────┐
│  projects   │ ──┬─ has many reviews
└─────────────┘   ├─ has verification_request
                  ├─ has dependencies
                  └─ may be featured

┌──────────────┐
│    reviews   │ ──┬─ belongs to project
└──────────────┘   └─ submitted by user

┌────────────────────────┐
│ verification_requests  │ ──┬─ for project
└────────────────────────┘   └─ requested by user / approved by admin

┌─────────────────────────┐
│ project_dependencies    │ ──┬─ project depends on other project
└─────────────────────────┘   └─ forms directed graph

┌──────────────────┐
│ featured_projects│ ──┬─ project featured by admin
└──────────────────┘   └─ curated by user (admin)
```

## Query Patterns

### Common Queries

**List projects with aggregated data**:
```sql
SELECT 
  p.id, p.name, p.owner, p.category, p.verified,
  p.rating, p.review_count,
  COALESCE(c.featured_at, NULL) as featured_at
FROM projects p
LEFT JOIN featured_projects c ON p.id = c.project_id
WHERE p.is_active = TRUE
  AND p.category = 'defi'
  AND p.verified = TRUE
ORDER BY p.rating DESC, p.review_count DESC
LIMIT 20 OFFSET 0;
```

**Full-text search**:
```sql
SELECT * FROM projects
WHERE to_tsvector('english', name || ' ' || COALESCE(description, '')) 
      @@ plainto_tsquery('english', 'stellar payment')
AND is_active = TRUE
ORDER BY ts_rank(to_tsvector('english', name), plainto_tsquery('english', 'stellar payment')) DESC
LIMIT 20;
```

**Get project with reviews**:
```sql
SELECT 
  p.*,
  json_agg(json_build_object(
    'id', r.id,
    'reviewer', r.reviewer,
    'rating', r.rating,
    'comment', r.comment,
    'created_at', r.created_at
  )) as reviews
FROM projects p
LEFT JOIN reviews r ON p.id = r.project_id AND r.hidden = FALSE
WHERE p.id = $1
GROUP BY p.id;
```

**Get user stats**:
```sql
SELECT 
  u.address,
  u.display_name,
  u.reputation_score,
  COUNT(DISTINCT p.id) as project_count,
  COUNT(DISTINCT r.id) as review_count
FROM users u
LEFT JOIN projects p ON u.address = p.owner AND p.is_active = TRUE
LEFT JOIN reviews r ON u.address = r.reviewer
WHERE u.address = $1
GROUP BY u.address;
```

## Indexing Strategy

### Indexes for Performance

| Table | Columns | Reason |
|-------|---------|--------|
| `projects` | `(owner)` | List projects by owner |
| `projects` | `(category)` | Filter by category |
| `projects` | `(verified)` | Filter verified projects |
| `projects` | `(rating DESC)` | Sort by rating |
| `projects` | `(created_at DESC)` | Sort by newest |
| `projects` | `name_gin` (GIN) | Full-text search on name |
| `reviews` | `(project_id)` | Get reviews for project |
| `reviews` | `(reviewer)` | Get user's reviews |
| `reviews` | `(rating)` | Filter by rating |
| `verification_requests` | `(status, created_at)` | Admin dashboards |
| `audit_logs` | `(timestamp DESC)` | View recent actions |

## Schema Evolution

### Migrations

New fields are added via migrations in `src/database/migrations/`. Each migration:

1. Has a timestamp filename: `YYYYMMDDHHMMSS_DescriptiveTitle.ts`
2. Includes `up()` and `down()` functions
3. Is reversible (down can undo up)

**Example**:
```typescript
// 20241106_AddCategoryFiltering.ts
export async function up(db: Database): Promise<void> {
  await db.query(`
    ALTER TABLE projects ADD COLUMN IF NOT EXISTS category VARCHAR(50);
    CREATE INDEX idx_projects_category ON projects(category);
  `);
}

export async function down(db: Database): Promise<void> {
  await db.query(`
    DROP INDEX IF EXISTS idx_projects_category;
    ALTER TABLE projects DROP COLUMN IF EXISTS category;
  `);
}
```

## Constraints & Data Integrity

### Unique Constraints

- `projects(contract_id)`: Each on-chain contract has one DB record
- `reviews(project_id, reviewer)`: One review per user per project
- `verification_requests(project_id)`: One active verification per project
- `project_dependencies(project_id, depends_on_project_id)`: No duplicate dependencies

### Foreign Key Constraints

- `reviews.project_id → projects.id` (ON DELETE CASCADE)
- `verification_requests.project_id → projects.id` (ON DELETE CASCADE)
- `project_dependencies.project_id → projects.id` (ON DELETE CASCADE)
- `project_dependencies.depends_on_project_id → projects.id` (ON DELETE CASCADE)

### Check Constraints

- `reviews(rating >= 1 AND rating <= 5)`: Valid rating range
- `project_dependencies(project_id != depends_on_project_id)`: No self-dependencies

## Backup & Recovery

### Backup Strategy

```bash
# Full database backup
pg_dump dongle_prod > dongle_backup_$(date +%Y%m%d).sql

# Point-in-time recovery (with WAL archiving enabled)
pg_basebackup -D ./backup -Ft -z -Xs -P
```

### Recovery Process

1. Restore from backup
2. Replay WAL if point-in-time recovery needed
3. Verify data integrity
4. Run migrations if schema has evolved
5. Restart application

## Performance Considerations

### Query Optimization

- **Pagination**: Always use LIMIT/OFFSET for large result sets
- **Selective columns**: SELECT only needed fields, not *
- **Indexing**: Create indexes on WHERE, JOIN, and ORDER BY columns
- **Aggregation**: Use database aggregation (COUNT, AVG) instead of application code
- **Denormalization**: Store `rating` and `review_count` on projects for fast access

### Maintenance

```bash
# Analyze query plans
EXPLAIN ANALYZE SELECT * FROM projects WHERE category = 'defi';

# Reindex for performance
REINDEX TABLE projects;

# Vacuum to reclaim space
VACUUM projects;
```

---

**See** [docs/ARCHITECTURE.md](ARCHITECTURE.md) for how services interact with these tables.
