# Audit Log REST API Design

## Overview
Development of a REST API route `/audit-logs` that offers extensive querying options for audit entries, including security restrictions based on user roles and permissions.

## Architecture

### 1. Core Components

#### **Controller Layer** (`packages/core/audit-log/server/src/controllers/audit-log.ts`)
- **Role**: Manages incoming HTTP requests & format of outgoing responses
- **Primary Function**: `find()` — Core request handler
- **Highlights**:
    - Cleans and validates incoming query parameters
    - Converts query input into filter structures
    - Supports pagination and sorting
    - Centralized error dispatching

#### **Service Layer** (`packages/core/audit-log/server/src/services/audit-log.ts`)
- **Role**: Houses business rules and data operations
- **Core Functions**:
    - `logContentApiOperation()` — Writes new audit entries
    - `getRecordAuditLogs()` — Retrieves logs scoped to a record
    - `getContentTypeAuditLogs()` — Retrieves logs scoped to a content type
    - `getUserAuditLogs()` — Retrieves logs scoped to a specific user
    - `getAuditStats()` — Aggregates analytics

#### **Middleware Layer** (`packages/core/audit-log/server/src/middlewares/content-api-audit.ts`)
- **Role**: Automatically generates audit records for Content API changes
- **Key Capabilities**:
    - Hooks into lifecycle of Content API routes
    - Captures metadata like request details and context
    - Logs creation, modification, deletion actions
    - Supports failure logging

#### **Policy Layer** (`packages/core/audit-log/server/src/policies/audit-log.ts`)
- **Role**: Handles access rights validation
- **Policy Checks**:
    - `canReadAuditLogs`
    - `canWriteAuditLogs`
    - `canAdminAuditLogs`
    - `isAuditLoggingEnabled`

### 2. Data Structure

#### **Audit Log Schema** (`packages/core/audit-log/server/src/content-types/audit-log/schema.json`)
```json
{
  "contentType": "string",
  "recordId": "string",
  "action": "enum[create,update,delete]",
  "timestamp": "datetime",
  "user": "relation",
  "changedFields": "json",
  "fullPayload": "json",
  "previousData": "json",
  "ipAddress": "string",
  "userAgent": "string",
  "requestId": "string",
  "metadata": "json"
}
```

#### **Database Indexes** (`packages/core/audit-log/server/src/migrations/20241201000000-add-audit-log-indexes.js`)
Improves read performance by indexing:
- `content_type + timestamp`
- `record_id + timestamp`
- `user + timestamp`
- `action + timestamp`
- `request_id`
- `timestamp` (for cleanup routines)

### 3. API Endpoint Design

#### **Main Route**: `/api/audit-logs`

#### **Available Methods**:
- `GET /api/audit-logs`
- `GET /api/audit-logs/:id`
- `GET /api/audit-logs/record/:contentType/:recordId`
- `GET /api/audit-logs/content-type/:contentType`
- `GET /api/audit-logs/user/:userId`
- `GET /api/audit-logs/stats`
- `POST /api/audit-logs/cleanup`

### 4. Filter System

#### **Query Processing**
```typescript
const sanitizedQuery = validateAndSanitizeQuery(ctx.query);
const filters = buildFilters(sanitizedQuery);
```

#### **Filter Capabilities**:

| Filter Type | Query Param | Example | Logic |
|------------|-------------|---------|------|
| Content Type | `contentType` | `?contentType=articles` | Exact match |
| User | `userId` | `?userId=123` | Numeric match |
| Action | `action` | `?action=update` | Enum comparison |
| Date Window | `startDate`, `endDate` | `?startDate=2024-01-01&endDate=2024-10-10` | Timestamp range |

Additional supported fields:
- `recordId`
- `requestId`
- `ipAddress`
- `userAgent`

### 5. Pagination & Sorting

#### Pagination Inputs:
- `page` (default: 1)
- `pageSize` (default: 25, max: 100)

#### Sorting Format:
Example:
```
?sort=timestamp:desc
```
Sortable fields: `timestamp`, `contentType`, `action`, `user`

### 6. API Responses

#### Sample Successful Response
```json
{
  "data": [
    {
      "id": 1,
      "contentType": "articles",
      "recordId": "123",
      "action": "create",
      "timestamp": "2024-01-01T10:00:00Z",
      "user": {
        "id": 1,
        "username": "admin",
        "email": "admin@example.com"
      },
      "changedFields": ["title", "content"],
      "ipAddress": "192.168.1.1",
      "userAgent": "Mozilla/5.0...",
      "requestId": "req_123456789",
      "metadata": {
        "method": "POST",
        "path": "/api/articles",
        "statusCode": 201,
        "responseTime": 150
      }
    }
  ],
  "meta": {
    "pagination": {
      "page": 1,
      "pageSize": 25,
      "pageCount": 10,
      "total": 250,
      "hasNextPage": true,
      "hasPrevPage": false
    },
    "filters": {
      "contentType": "articles",
      "userId": null,
      "action": null,
      "startDate": "2024-01-01",
      "endDate": "2024-12-31"
    },
    "sort": ["timestamp:desc"]
  }
}
```

### 7. Security Enforcement

#### Requirements:
- Admin authentication enforced
- JWT validation mandatory

#### Permission Keys:
- `read_audit_logs` → GET endpoints
- `write_audit_logs` → POST endpoints
- `admin_audit_logs` → privileged routes

#### Rate Limits:
- Per user & per IP rules
- Configurable quotas

### 8. Configuration Options

#### Default Settings File
```typescript
{
  enabled: true,
  excludeContentTypes: ['audit-log', 'strapi::core-store'],
  excludedFields: ['id', 'createdAt', 'updatedAt'],
  logLevels: ['create', 'update', 'delete'],
  permissions: {
    readPermission: 'plugin::audit-log.read_audit_logs',
    writePermission: 'plugin::audit-log.write_audit_logs',
    adminPermission: 'plugin::audit-log.admin_audit_logs'
  }
}
```

### 9. Performance Enhancements

- Strategic indexes to improve frequent lookups
- Efficient pagination strategy
- Selective population of relational fields
- Optional caching layer with TTL-based invalidation

### 10. Error Responses

#### Status Codes:
- `200` → OK
- `400` → Invalid request input
- `401` → Unauthorized session
- `403` → Permission denied
- `404` → Resource missing
- `503` → Audit logging turned off

#### Format:
```json
{
  "error": {
    "status": 400,
    "name": "ValidationError",
    "message": "Invalid query parameters",
    "details": {
      "contentType": ["Invalid content type format"]
    }
  }
}
```

### 11. Execution Pipeline

1. Receive request
2. Validate user identity
3. Check authorization rules
4. Scrutinize query parameters
5. Compile DB filter conditions
6. Retrieve matching records
7. Wrap data into correct response structure
8. Send back to client

### 12. Testing Plan

#### Unit-Level
- Controllers
- Services
- Authorization policies

#### End-to-End
- Data access
- API surface behavior
- Authenticated access scenarios

#### Load & Stability
- Stress queries with large datasets
- DB performance checks
- Memory load verification

---
