# Dongle Backend — API Reference

## Overview

The Dongle Backend API provides RESTful endpoints for managing projects, reviews, verification requests, and user profiles. All endpoints return JSON responses and support standardized error handling.

**Base URL**: `http://localhost:3001/api` (development) or `https://api.dongle.app` (production)

**Authentication**: Bearer token (JWT) required for write operations. Include in `Authorization` header:
```
Authorization: Bearer <jwt_token>
```

## Response Format

All responses follow a standard format:

**Success Response (2xx)**:
```json
{
  "data": { /* response payload */ },
  "meta": {
    "timestamp": "2024-11-06T10:30:00Z",
    "requestId": "req_abc123"
  }
}
```

**List Response with Pagination**:
```json
{
  "data": [ /* items */ ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 156,
    "hasMore": true
  },
  "meta": { "timestamp": "...", "requestId": "..." }
}
```

**Error Response (4xx/5xx)**:
```json
{
  "error": {
    "code": "INVALID_PROJECT",
    "message": "Project not found",
    "status": 404,
    "details": { /* optional: field-level errors */ }
  },
  "meta": { "timestamp": "...", "requestId": "..." }
}
```

## Projects Endpoints

### List Projects

**Endpoint**: `GET /projects`

**Query Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `page` | integer | No | Page number (default: 1) |
| `limit` | integer | No | Items per page (default: 20, max: 100) |
| `search` | string | No | Full-text search in name/description |
| `category` | string | No | Filter by category (defi, nft, payment, etc.) |
| `verified` | boolean | No | Filter by verification status |
| `owner` | string | No | Filter by owner address |
| `sort` | string | No | Sort field: `created_at`, `updated_at`, `rating`, `review_count` |
| `order` | string | No | Sort order: `asc`, `desc` (default: `desc`) |

**Example Request**:
```bash
curl "http://localhost:3001/api/projects?search=stellar&category=defi&verified=true&limit=20"
```

**Response**:
```json
{
  "data": [
    {
      "id": "proj_123",
      "name": "Stellar Payment Hub",
      "description": "Fast payments on Stellar",
      "owner": "GXXXXX",
      "website": "https://...",
      "category": "payment",
      "verified": true,
      "rating": 4.5,
      "reviewCount": 42,
      "metadataCid": "QmXXX...",
      "createdAt": "2024-10-01T10:00:00Z",
      "updatedAt": "2024-11-01T10:00:00Z"
    }
    /* ... more projects */
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 156,
    "hasMore": true
  }
}
```

**Status Codes**:
- `200 OK` — Successfully retrieved projects
- `400 Bad Request` — Invalid query parameters
- `500 Internal Server Error` — Server error

### Get Project by ID

**Endpoint**: `GET /projects/:projectId`

**Path Parameters**:
| Parameter | Type | Description |
|-----------|------|-------------|
| `projectId` | string | Unique project identifier |

**Example Request**:
```bash
curl http://localhost:3001/api/projects/proj_123
```

**Response**:
```json
{
  "data": {
    "id": "proj_123",
    "name": "Stellar Payment Hub",
    "description": "...",
    "owner": "GXXXXX",
    "website": "https://...",
    "category": "payment",
    "verified": true,
    "rating": 4.5,
    "reviewCount": 42,
    "metadataCid": "QmXXX...",
    "createdAt": "2024-10-01T10:00:00Z",
    "updatedAt": "2024-11-01T10:00:00Z",
    "reviews": [
      {
        "id": "rev_456",
        "reviewer": "GYYYYY",
        "rating": 5,
        "comment": "Excellent service!",
        "createdAt": "2024-11-05T12:00:00Z"
      }
    ]
  }
}
```

**Status Codes**:
- `200 OK` — Project found
- `404 Not Found` — Project doesn't exist
- `500 Internal Server Error` — Server error

### Create Project

**Endpoint**: `POST /projects`

**Authentication**: Required (Bearer token)

**Request Body**:
```json
{
  "name": "My Project",
  "description": "A description of my project",
  "website": "https://myproject.app",
  "category": "defi",
  "metadataCid": "QmXXX...",
  "owner": "GXXXXX"
}
```

**Validation Rules**:
- `name`: Required, 1-100 characters
- `description`: Optional, max 1000 characters
- `website`: Optional, valid URL format
- `category`: Required, one of: `defi`, `nft`, `payment`, `tools`, `other`
- `metadataCid`: Optional, valid IPFS CID format
- `owner`: Required, valid Stellar address format

**Example Request**:
```bash
curl -X POST http://localhost:3001/api/projects \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "My Project",
    "description": "A great project",
    "website": "https://myproject.app",
    "category": "defi",
    "owner": "GXXXXX"
  }'
```

**Response**:
```json
{
  "data": {
    "id": "proj_789",
    "name": "My Project",
    "description": "...",
    "owner": "GXXXXX",
    "category": "defi",
    "verified": false,
    "rating": null,
    "reviewCount": 0,
    "createdAt": "2024-11-06T10:30:00Z"
  }
}
```

**Status Codes**:
- `201 Created` — Project created successfully
- `400 Bad Request` — Invalid input data
- `401 Unauthorized` — Missing or invalid token
- `409 Conflict` — Project name already exists
- `500 Internal Server Error` — Server error

### Update Project

**Endpoint**: `PATCH /projects/:projectId`

**Authentication**: Required (Bearer token, must be project owner)

**Request Body** (all fields optional):
```json
{
  "name": "Updated Name",
  "description": "Updated description",
  "website": "https://updated.app",
  "metadataCid": "QmYYY..."
}
```

**Example Request**:
```bash
curl -X PATCH http://localhost:3001/api/projects/proj_123 \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Updated Name",
    "description": "New description"
  }'
```

**Response**:
```json
{
  "data": {
    "id": "proj_123",
    "name": "Updated Name",
    "description": "New description",
    "updatedAt": "2024-11-06T11:00:00Z"
  }
}
```

**Status Codes**:
- `200 OK` — Project updated successfully
- `400 Bad Request` — Invalid input data
- `401 Unauthorized` — Missing, invalid, or wrong owner token
- `404 Not Found` — Project doesn't exist
- `500 Internal Server Error` — Server error

### Get Project Statistics

**Endpoint**: `GET /projects/:projectId/stats`

**Example Request**:
```bash
curl http://localhost:3001/api/projects/proj_123/stats
```

**Response**:
```json
{
  "data": {
    "projectId": "proj_123",
    "totalReviews": 42,
    "averageRating": 4.5,
    "ratingDistribution": {
      "5": 30,
      "4": 8,
      "3": 3,
      "2": 1,
      "1": 0
    },
    "recentActivity": "2024-11-05T14:00:00Z",
    "verificationStatus": "APPROVED",
    "verifiedAt": "2024-10-01T10:00:00Z"
  }
}
```

**Status Codes**:
- `200 OK` — Statistics retrieved
- `404 Not Found` — Project doesn't exist

## Reviews Endpoints

### List Reviews for Project

**Endpoint**: `GET /projects/:projectId/reviews`

**Query Parameters**:
| Parameter | Type | Description |
|-----------|------|-------------|
| `page` | integer | Page number (default: 1) |
| `limit` | integer | Items per page (default: 20, max: 100) |
| `sort` | string | Sort by: `recent`, `helpful`, `rating` |

**Example Request**:
```bash
curl "http://localhost:3001/api/projects/proj_123/reviews?limit=20&sort=recent"
```

**Response**:
```json
{
  "data": [
    {
      "id": "rev_456",
      "projectId": "proj_123",
      "reviewer": "GYYYYY",
      "rating": 5,
      "comment": "Excellent service!",
      "helpful": 12,
      "createdAt": "2024-11-05T12:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 42,
    "hasMore": false
  }
}
```

### Submit Review

**Endpoint**: `POST /projects/:projectId/reviews`

**Authentication**: Required (Bearer token)

**Request Body**:
```json
{
  "rating": 4,
  "comment": "Great project, very easy to use",
  "reviewer": "GYYYYY"
}
```

**Validation Rules**:
- `rating`: Required, integer 1-5
- `comment`: Required, 10-1000 characters
- `reviewer`: Required, valid Stellar address

**Example Request**:
```bash
curl -X POST http://localhost:3001/api/projects/proj_123/reviews \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "rating": 4,
    "comment": "Great project!",
    "reviewer": "GYYYYY"
  }'
```

**Response**:
```json
{
  "data": {
    "id": "rev_789",
    "projectId": "proj_123",
    "reviewer": "GYYYYY",
    "rating": 4,
    "comment": "Great project!",
    "createdAt": "2024-11-06T10:30:00Z"
  }
}
```

**Status Codes**:
- `201 Created` — Review created successfully
- `400 Bad Request` — Invalid input data
- `401 Unauthorized` — Missing or invalid token
- `409 Conflict` — User already reviewed this project
- `404 Not Found` — Project doesn't exist

### Update Review

**Endpoint**: `PATCH /reviews/:reviewId`

**Authentication**: Required (Bearer token, must be reviewer)

**Request Body**:
```json
{
  "rating": 5,
  "comment": "Actually it's even better than I thought!"
}
```

**Example Request**:
```bash
curl -X PATCH http://localhost:3001/api/reviews/rev_456 \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "rating": 5,
    "comment": "Updated review"
  }'
```

**Status Codes**:
- `200 OK` — Review updated successfully
- `401 Unauthorized` — Not the reviewer
- `404 Not Found` — Review doesn't exist

### Delete Review

**Endpoint**: `DELETE /reviews/:reviewId`

**Authentication**: Required (Bearer token, must be reviewer or admin)

**Example Request**:
```bash
curl -X DELETE http://localhost:3001/api/reviews/rev_456 \
  -H "Authorization: Bearer <token>"
```

**Status Codes**:
- `204 No Content` — Review deleted successfully
- `401 Unauthorized` — Not authorized to delete
- `404 Not Found` — Review doesn't exist

## Verification Endpoints

### Request Verification

**Endpoint**: `POST /verification/request`

**Authentication**: Required (Bearer token)

**Request Body**:
```json
{
  "projectId": "proj_123",
  "submittedBy": "GXXXXX"
}
```

**Example Request**:
```bash
curl -X POST http://localhost:3001/api/verification/request \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "projectId": "proj_123",
    "submittedBy": "GXXXXX"
  }'
```

**Response**:
```json
{
  "data": {
    "id": "vreq_123",
    "projectId": "proj_123",
    "submittedBy": "GXXXXX",
    "status": "PENDING",
    "createdAt": "2024-11-06T10:30:00Z"
  }
}
```

**Status Codes**:
- `201 Created` — Verification request created
- `400 Bad Request` — Invalid input
- `409 Conflict` — Request already exists for this project

### Get Verification Status

**Endpoint**: `GET /verification/projects/:projectId/status`

**Example Request**:
```bash
curl http://localhost:3001/api/verification/projects/proj_123/status
```

**Response**:
```json
{
  "data": {
    "projectId": "proj_123",
    "status": "PENDING",
    "requestId": "vreq_123",
    "requestedAt": "2024-11-06T10:30:00Z",
    "approvedAt": null,
    "rejectedAt": null,
    "rejectionReason": null
  }
}
```

### List Verification Requests (Admin Only)

**Endpoint**: `GET /admin/verification/requests`

**Authentication**: Required (Bearer token, must be admin)

**Query Parameters**:
| Parameter | Type | Description |
|-----------|------|-------------|
| `status` | string | Filter by status: `PENDING`, `APPROVED`, `REJECTED` |
| `page` | integer | Page number (default: 1) |
| `limit` | integer | Items per page (default: 20, max: 100) |

**Example Request**:
```bash
curl "http://localhost:3001/api/admin/verification/requests?status=PENDING" \
  -H "Authorization: Bearer <admin_token>"
```

**Response**:
```json
{
  "data": [
    {
      "id": "vreq_123",
      "projectId": "proj_123",
      "projectName": "My Project",
      "submittedBy": "GXXXXX",
      "status": "PENDING",
      "createdAt": "2024-11-06T10:30:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 5,
    "hasMore": false
  }
}
```

### Approve Verification (Admin Only)

**Endpoint**: `POST /admin/verification/:projectId/approve`

**Authentication**: Required (Bearer token, must be admin)

**Request Body**:
```json
{
  "approvedBy": "GADMIN123",
  "notes": "Meets all criteria"
}
```

**Example Request**:
```bash
curl -X POST http://localhost:3001/api/admin/verification/proj_123/approve \
  -H "Authorization: Bearer <admin_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "approvedBy": "GADMIN123",
    "notes": "Meets all criteria"
  }'
```

**Status Codes**:
- `200 OK` — Verification approved
- `401 Unauthorized` — Not admin
- `404 Not Found` — Project or request not found

### Reject Verification (Admin Only)

**Endpoint**: `POST /admin/verification/:projectId/reject`

**Authentication**: Required (Bearer token, must be admin)

**Request Body**:
```json
{
  "rejectedBy": "GADMIN123",
  "reason": "Does not meet security requirements"
}
```

**Status Codes**:
- `200 OK` — Verification rejected
- `401 Unauthorized` — Not admin
- `404 Not Found` — Project or request not found

## User Endpoints

### Get User Profile

**Endpoint**: `GET /users/:address`

**Example Request**:
```bash
curl http://localhost:3001/api/users/GXXXXX
```

**Response**:
```json
{
  "data": {
    "address": "GXXXXX",
    "displayName": "John Developer",
    "bio": "Stellar enthusiast",
    "avatarUrl": "https://...",
    "reputationScore": 42.5,
    "projectCount": 3,
    "reviewCount": 12,
    "createdAt": "2024-10-01T10:00:00Z"
  }
}
```

### Update User Profile

**Endpoint**: `PATCH /users/:address`

**Authentication**: Required (Bearer token, must be user)

**Request Body**:
```json
{
  "displayName": "Updated Name",
  "bio": "Updated bio",
  "avatarUrl": "https://..."
}
```

**Status Codes**:
- `200 OK` — Profile updated
- `401 Unauthorized` — Not authorized
- `404 Not Found` — User not found

### Get User's Projects

**Endpoint**: `GET /users/:address/projects`

**Query Parameters**: `page`, `limit`

**Response**:
```json
{
  "data": [
    { /* project objects */ }
  ],
  "pagination": { /* pagination info */ }
}
```

### Get User's Reviews

**Endpoint**: `GET /users/:address/reviews`

**Query Parameters**: `page`, `limit`

**Response**:
```json
{
  "data": [
    { /* review objects */ }
  ],
  "pagination": { /* pagination info */ }
}
```

## Health & Admin Endpoints

### Health Check

**Endpoint**: `GET /health`

**No Authentication Required**

**Response**:
```json
{
  "data": {
    "status": "ok",
    "timestamp": "2024-11-06T10:30:00Z"
  }
}
```

### Readiness Check

**Endpoint**: `GET /health/ready`

**Response**:
```json
{
  "data": {
    "database": "ok",
    "sorobanRpc": "ok",
    "indexer": "ok"
  }
}
```

### Admin System Health

**Endpoint**: `GET /admin/health`

**Authentication**: Required (Bearer token, must be admin)

**Response**:
```json
{
  "data": {
    "uptime": 3600000,
    "database": {
      "connected": true,
      "latency": 12,
      "poolSize": 10
    },
    "indexer": {
      "running": true,
      "lag": 2,
      "lastSync": "2024-11-06T10:29:00Z"
    }
  }
}
```

### Indexer Status

**Endpoint**: `GET /admin/indexer/status`

**Authentication**: Required (Bearer token, must be admin)

**Response**:
```json
{
  "data": {
    "running": true,
    "currentLedger": 1234567,
    "lastSync": "2024-11-06T10:29:00Z",
    "lag": 2,
    "eventsProcessed": 1523,
    "errors": 0
  }
}
```

---

## Error Codes Reference

| Code | Status | Description |
|------|--------|-------------|
| `INVALID_PROJECT` | 404 | Project not found |
| `INVALID_REVIEW` | 404 | Review not found |
| `UNAUTHORIZED` | 401 | Missing or invalid authentication token |
| `FORBIDDEN` | 403 | Not authorized to perform this action |
| `VALIDATION_ERROR` | 400 | Invalid input data |
| `DUPLICATE_REVIEW` | 409 | User already reviewed this project |
| `DATABASE_ERROR` | 500 | Database operation failed |
| `SOROBAN_ERROR` | 500 | Smart contract interaction failed |
| `RATE_LIMITED` | 429 | Too many requests, please try again later |

---

**See** [docs/ARCHITECTURE.md](ARCHITECTURE.md) for system design details.
