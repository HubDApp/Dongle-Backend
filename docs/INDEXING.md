# Dongle Backend — Indexing & Event Sync

## Overview

The indexing service continuously monitors Soroban smart contract events and synchronizes them with the PostgreSQL database. This ensures the backend's off-chain data stays in sync with on-chain state while enabling fast queries, aggregation, and analytics that Soroban cannot perform.

## Architecture

```
Soroban Smart Contract (Stellar)
         ↓ (emits events)
Event Stream (Soroban RPC)
         ↓
Indexer Service
├─ Fetch ledgers
├─ Parse events
└─ Process changes
         ↓
PostgreSQL Database
├─ projects table
├─ reviews table
├─ verification_requests table
└─ audit_logs table
         ↓
Frontend API
└─ Fast queries with aggregation
```

## Event Types

### ProjectRegistered

**Emitted by**: Smart contract `register_project()` call

**Data**:
```json
{
  "topic": "ProjectRegistered",
  "data": {
    "project_id": "uuid",
    "owner": "GXXXXX",
    "name": "Project Name",
    "description": "...",
    "website": "https://...",
    "category": "defi",
    "metadata_cid": "QmXXX..."
  }
}
```

**Handler**:
```typescript
async function handleProjectRegistered(event) {
  // Check if project exists
  const existing = await projectRepository.findByContractId(event.data.project_id);
  
  if (!existing) {
    // Create new project
    await projectRepository.create({
      contract_id: event.data.project_id,
      owner: event.data.owner,
      name: event.data.name,
      // ... other fields
    });
    
    // Create/update user profile
    await userRepository.incrementProjectCount(event.data.owner);
  }
  
  // Log audit event
  await auditRepository.log({
    action: 'project_registered',
    subject_type: 'project',
    subject_id: event.data.project_id,
    changes: event.data
  });
}
```

### ProjectUpdated

**Emitted by**: Smart contract `update_project()` call

**Data**:
```json
{
  "topic": "ProjectUpdated",
  "data": {
    "project_id": "uuid",
    "name": "New Name",
    "description": "New description",
    "website": "https://newsite.com",
    "metadata_cid": "QmYYY..."
  }
}
```

**Handler**:
```typescript
async function handleProjectUpdated(event) {
  const project = await projectRepository.findByContractId(event.data.project_id);
  
  if (project) {
    // Update only provided fields
    const updates = Object.fromEntries(
      Object.entries(event.data).filter(([k, v]) => v !== null && v !== undefined)
    );
    
    await projectRepository.update(project.id, updates);
    
    // Cache invalidation
    await cacheService.invalidate(`project:${project.id}`);
  }
}
```

### ReviewSubmitted

**Emitted by**: Smart contract `submit_review()` call

**Data**:
```json
{
  "topic": "ReviewSubmitted",
  "data": {
    "review_id": "uuid",
    "project_id": "uuid",
    "reviewer": "GYYYYY",
    "rating": 4,
    "comment": "Great project!",
    "metadata_cid": "QmZZZ..."
  }
}
```

**Handler**:
```typescript
async function handleReviewSubmitted(event) {
  // Create review
  await reviewRepository.create({
    id: event.data.review_id,
    project_id: event.data.project_id,
    reviewer: event.data.reviewer,
    rating: event.data.rating,
    comment: event.data.comment,
    metadata_cid: event.data.metadata_cid
  });
  
  // Recalculate project rating
  const stats = await reviewRepository.getProjectStats(event.data.project_id);
  await projectRepository.update(event.data.project_id, {
    rating: stats.averageRating,
    review_count: stats.count
  });
  
  // Update user profile
  await userRepository.incrementReviewCount(event.data.reviewer);
  
  // Invalidate caches
  await cacheService.invalidate(`project:${event.data.project_id}:reviews`);
  await cacheService.invalidate(`user:${event.data.reviewer}:reviews`);
}
```

### VerificationRequested

**Emitted by**: Smart contract `request_verification()` call

**Data**:
```json
{
  "topic": "VerificationRequested",
  "data": {
    "request_id": "uuid",
    "project_id": "uuid",
    "requested_by": "GXXXXX"
  }
}
```

**Handler**:
```typescript
async function handleVerificationRequested(event) {
  // Create verification request
  await verificationRepository.create({
    id: event.data.request_id,
    project_id: event.data.project_id,
    requested_by: event.data.requested_by,
    status: 'PENDING'
  });
  
  // Update project status
  await projectRepository.update(event.data.project_id, {
    verification_status: 'PENDING'
  });
  
  // Send notification to admins
  await notificationService.notifyAdmins({
    type: 'verification_request',
    project_id: event.data.project_id,
    requested_by: event.data.requested_by
  });
}
```

### VerificationApproved

**Emitted by**: Smart contract `approve_verification()` call

**Data**:
```json
{
  "topic": "VerificationApproved",
  "data": {
    "project_id": "uuid",
    "approved_by": "GADMIN123",
    "notes": "Meets all criteria"
  }
}
```

**Handler**:
```typescript
async function handleVerificationApproved(event) {
  // Update verification request
  await verificationRepository.update(event.data.project_id, {
    status: 'APPROVED',
    approved_by: event.data.approved_by,
    approved_at: new Date()
  });
  
  // Mark project as verified
  await projectRepository.update(event.data.project_id, {
    verified: true,
    verification_status: 'APPROVED',
    verified_at: new Date()
  });
  
  // Notify project owner
  const project = await projectRepository.findByContractId(event.data.project_id);
  await notificationService.notifyUser(project.owner, {
    type: 'verification_approved',
    project_id: event.data.project_id
  });
  
  // Invalidate caches
  await cacheService.invalidate(`project:${event.data.project_id}`);
  await cacheService.invalidate('projects:featured');
}
```

### VerificationRejected

**Emitted by**: Smart contract `reject_verification()` call

**Data**:
```json
{
  "topic": "VerificationRejected",
  "data": {
    "project_id": "uuid",
    "rejected_by": "GADMIN123",
    "reason": "Security concerns"
  }
}
```

**Handler**: Similar to VerificationApproved, updates status to REJECTED

## Indexer Service Flow

### Initialization

```typescript
class IndexerService {
  async start() {
    // Get last synced ledger from database
    const state = await indexerRepository.getState();
    this.currentLedger = state?.lastSyncedLedger || STARTING_LEDGER;
    
    // Start polling loop
    this.isRunning = true;
    this.pollLoop();
  }
  
  private async pollLoop() {
    while (this.isRunning) {
      try {
        const newEvents = await this.syncEvents();
        
        if (newEvents.length > 0) {
          logger.info(`Processed ${newEvents.length} events`);
        }
        
        // Sleep before next poll
        await sleep(POLL_INTERVAL);
      } catch (error) {
        logger.error('Indexer error', error);
        // Continue polling even on error
      }
    }
  }
}
```

### Event Fetching

```typescript
async function syncEvents() {
  // Fetch events from Soroban RPC
  const response = await sorobanRpc.getEvents({
    startLedger: this.currentLedger,
    filters: [
      {
        contractId: PROJECT_REGISTRY_CONTRACT,
        topics: [
          'ProjectRegistered',
          'ProjectUpdated',
          'ReviewSubmitted',
          'VerificationRequested',
          'VerificationApproved'
        ]
      }
    ]
  });
  
  const events = response.events;
  
  if (events.length > 0) {
    // Process events in transaction
    await this.processEventBatch(events);
    
    // Update last synced ledger
    this.currentLedger = Math.max(
      ...events.map(e => e.ledger)
    ) + 1;
  }
  
  return events;
}
```

### Event Processing

```typescript
async function processEventBatch(events: Event[]) {
  // Start database transaction
  const trx = await db.transaction();
  
  try {
    for (const event of events) {
      await this.processEvent(event, trx);
    }
    
    // Commit transaction
    await trx.commit();
    
    // Update indexer state
    await indexerRepository.updateState({
      lastSyncedLedger: Math.max(...events.map(e => e.ledger)),
      lastSyncedAt: new Date(),
      eventsProcessed: events.length
    });
    
  } catch (error) {
    // Rollback on error
    await trx.rollback();
    throw error;
  }
}

async function processEvent(event: Event, trx: Transaction) {
  const handler = this.getEventHandler(event.topic);
  
  if (!handler) {
    logger.warn(`No handler for event: ${event.topic}`);
    return;
  }
  
  // Parse event data
  const parsed = this.parseEventData(event);
  
  // Call handler
  await handler(parsed, trx);
  
  // Log successful processing
  logger.debug(`Processed event: ${event.topic}`, {
    eventId: event.id,
    ledger: event.ledger
  });
}
```

## Recovery & Resync

### Gap Detection

```typescript
async function detectGaps() {
  const response = await sorobanRpc.getLatestLedger();
  const latestLedger = response.sequence;
  
  const indexerState = await indexerRepository.getState();
  const lag = latestLedger - indexerState.lastSyncedLedger;
  
  if (lag > ACCEPTABLE_LAG) {
    logger.warn(`Indexer lag exceeds threshold: ${lag} ledgers`);
    
    // Emit alert
    await monitoring.alert({
      severity: 'warning',
      message: `Indexer lag: ${lag} ledgers`,
      threshold: ACCEPTABLE_LAG
    });
  }
}
```

### Manual Resync

**Endpoint**: `POST /api/admin/indexer/resync?startLedger=1000000`

**Use cases**:
- Bug fix required reprocessing historical events
- New event handler added
- Indexer missed events due to downtime

**Process**:
```typescript
async function resync(startLedger: number) {
  logger.info(`Starting resync from ledger ${startLedger}`);
  
  // Reset indexer state
  await indexerRepository.updateState({
    lastSyncedLedger: startLedger - 1,
    resyncInProgress: true
  });
  
  // Wait for full resync
  const finalLedger = await this.pollUntilCurrent();
  
  // Mark complete
  await indexerRepository.updateState({
    resyncInProgress: false,
    resyncCompletedAt: new Date()
  });
  
  logger.info(`Resync complete. Final ledger: ${finalLedger}`);
  
  return { startLedger, finalLedger };
}
```

## Monitoring & Observability

### Indexer Status

**Endpoint**: `GET /api/admin/indexer/status`

**Response**:
```json
{
  "running": true,
  "currentLedger": 1234567,
  "lastSync": "2024-11-06T10:29:00Z",
  "lag": 2,
  "eventsProcessed": 1523,
  "errors": 0,
  "uptime": 3600000
}
```

### Key Metrics

| Metric | Description | Target | Alert |
|--------|-------------|--------|-------|
| **Lag** | Ledgers behind latest | < 1 | > 10 |
| **Events/minute** | Processing throughput | 100-500 | < 10 |
| **Error rate** | Failed event processing | 0% | > 0.1% |
| **Poll time** | Time to fetch ledger | < 1s | > 5s |
| **Transaction time** | Time to process batch | < 2s | > 10s |

### Logging

**Event processing logs**:
```json
{
  "timestamp": "2024-11-06T10:30:00Z",
  "level": "info",
  "service": "IndexerService",
  "action": "processEvent",
  "eventType": "ReviewSubmitted",
  "projectId": "proj_123",
  "ledger": 1234567,
  "duration": 245
}
```

## Error Handling

### Common Errors

| Error | Cause | Recovery |
|-------|-------|----------|
| **Event not found** | Ledger pruned by Soroban | Resume from latest available |
| **Parse error** | Malformed event data | Log and skip, continue |
| **Database error** | Connection lost, disk full | Retry with backoff, alert ops |
| **Rate limit** | Too many RPC calls | Reduce poll frequency, wait |
| **Contract call failed** | Invalid contract ID | Check configuration, restart |

### Retry Strategy

```typescript
async function executeWithRetry(
  fn: () => Promise<T>,
  maxRetries: number = 3
): Promise<T> {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      
      // Exponential backoff: 1s, 2s, 4s
      const delay = Math.pow(2, i) * 1000;
      await sleep(delay);
      
      logger.warn(`Retry ${i + 1}/${maxRetries} after ${delay}ms`, error);
    }
  }
}
```

## Performance Optimization

### Batch Processing

```typescript
// Process events in batches of 100
const BATCH_SIZE = 100;

for (let i = 0; i < events.length; i += BATCH_SIZE) {
  const batch = events.slice(i, i + BATCH_SIZE);
  await processBatch(batch);
  
  // Flush cache every batch
  await cache.flush();
}
```

### Query Optimization

```typescript
// Use bulk insert for multiple reviews
async function bulkInsertReviews(reviews: Review[]) {
  const query = `
    INSERT INTO reviews (project_id, reviewer, rating, comment)
    VALUES ${reviews.map((_, i) => `($${i * 4 + 1}, $${i * 4 + 2}, $${i * 4 + 3}, $${i * 4 + 4})`).join(', ')}
  `;
  
  const values = reviews.flatMap(r => [r.project_id, r.reviewer, r.rating, r.comment]);
  
  return await db.query(query, values);
}
```

## Testing Indexer

### Unit Tests

```typescript
describe('IndexerService', () => {
  describe('handleReviewSubmitted', () => {
    it('should create review and update rating', async () => {
      const event = {
        topic: 'ReviewSubmitted',
        data: { projectId, rating: 5, comment: '...' }
      };
      
      await indexer.handleReviewSubmitted(event);
      
      const review = await reviewRepository.findById(...);
      expect(review.rating).toBe(5);
      
      const project = await projectRepository.findById(projectId);
      expect(project.rating).toBe(5);
    });
  });
});
```

### Integration Tests

```typescript
describe('Indexer Integration', () => {
  it('should sync events from Soroban', async () => {
    // Mock Soroban RPC response
    mockSorobanRpc.getEvents.mockResolvedValue({
      events: [
        { topic: 'ProjectRegistered', data: {...} },
        { topic: 'ReviewSubmitted', data: {...} }
      ]
    });
    
    // Start indexer
    await indexer.start();
    await sleep(1000);
    
    // Verify database state
    const projects = await projectRepository.list();
    expect(projects).toHaveLength(1);
    
    const reviews = await reviewRepository.list();
    expect(reviews).toHaveLength(1);
  });
});
```

## Future Enhancements

- [ ] **Event streaming**: Replace polling with streaming (WebSocket)
- [ ] **Parallel processing**: Process multiple events concurrently
- [ ] **Event replay**: Reprocess historical events for testing
- [ ] **Indexer health dashboard**: Real-time monitoring UI
- [ ] **Schema evolution**: Automatic migration on contract upgrades

---

**See** [docs/ARCHITECTURE.md](ARCHITECTURE.md) for overall system design.
