# API Contract: RiskWeb Core v1

**Branch**: `001-risk-web-core` | **Date**: 2026-03-12

This document defines the REST API contract for the RiskWeb Fund Account Management system per Constitution Article VI.

---

## Base URL

```
https://riskweb.internal/api/v1
```

---

## Authentication

All endpoints require Kerberos/SPNEGO authentication. The browser negotiates a Kerberos ticket transparently. No login page or form exists.

**Headers**:
```
Authorization: Negotiate <base64-encoded-token>
```

**Response on failure** (HTTP 401):
```json
{
  "data": null,
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Kerberos authentication failed. Please ensure you are on the corporate network."
  }
}
```

---

## Response Envelope

All responses use a consistent envelope format:

### Success Response
```json
{
  "data": { ... },
  "error": null
}
```

### Error Response
```json
{
  "data": null,
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error description"
  }
}
```

---

## Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| UNAUTHORIZED | 401 | Kerberos authentication failed |
| FORBIDDEN | 403 | Action not allowed (e.g., maker approving own request) |
| NOT_FOUND | 404 | Resource does not exist |
| CONFLICT | 409 | Conflicting state (e.g., request already approved) |
| VALIDATION_ERROR | 400 | Request validation failed |
| INTERNAL_ERROR | 500 | Unexpected server error |

---

## Endpoints

### 1. List Accounts (Paginated)

```
GET /accounts
```

Returns paginated list of Fund Accounts. Defaults to active accounts only.

**Query Parameters**:

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| page | integer | No | 0 | Page number (0-indexed) |
| size | integer | No | 20 | Page size (10-50) |
| search | string | No | "" | Search term for Fund Name, Fund Type, Short Name |
| sort | string | No | "fundName,asc" | Sort field and direction |
| includeInactive | boolean | No | false | Include inactive accounts |

**Response** (HTTP 200):
```json
{
  "data": {
    "content": [
      {
        "fundId": "ABC123",
        "fundName": "AB Equity Fund",
        "epaceId": "EP-001",
        "shortName": "Equity",
        "fundType": "Equity",
        "reportingFrequency": "Monthly",
        "esma40Act": "ESMA",
        "rmAcronym": "EQ",
        "activeStatus": "ACTIVE",
        "createdBy": "CORPORATE\\jdoe",
        "createdOn": "2026-01-15T10:30:00Z"
      }
    ],
    "page": 0,
    "size": 20,
    "totalElements": 150,
    "totalPages": 8
  },
  "error": null
}
```

---

### 2. Get Single Account

```
GET /accounts/{fundId}
```

Returns a single Fund Account by its Fund ID.

**Path Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| fundId | string | The Fund ID (primary key) |

**Response** (HTTP 200):
```json
{
  "data": {
    "fundId": "ABC123",
    "fundName": "AB Equity Fund",
    "epaceId": "EP-001",
    "shortName": "Equity",
    "fundType": "Equity",
    "reportingFrequency": "Monthly",
    "esma40Act": "ESMA",
    "rmAcronym": "EQ",
    "activeStatus": "ACTIVE",
    "createdBy": "CORPORATE\\jdoe",
    "createdOn": "2026-01-15T10:30:00Z"
  },
  "error": null
}
```

**Error** (HTTP 404):
```json
{
  "data": null,
  "error": {
    "code": "NOT_FOUND",
    "message": "Account with Fund ID 'XYZ999' not found"
  }
}
```

---

### 3. Submit CREATE Change Request

```
POST /accounts/requests
```

Submits a new account for approval (maker action).

**Request Body**:
```json
{
  "operationType": "CREATE",
  "accountData": {
    "fundId": "DEF456",
    "fundName": "AB Fixed Income Fund",
    "epaceId": "EP-002",
    "shortName": "FixedIncome",
    "fundType": "Fixed Income",
    "reportingFrequency": "Quarterly",
    "esma40Act": "40ACT",
    "rmAcronym": "FI"
  }
}
```

**Response** (HTTP 201):
```json
{
  "data": {
    "requestId": 42,
    "operationType": "CREATE",
    "accountId": null,
    "snapshot": { ... },
    "makerLogin": "CORPORATE\\jdoe",
    "makerTimestamp": "2026-03-12T14:30:00Z",
    "status": "PENDING_APPROVAL"
  },
  "error": null
}
```

**Validation Error** (HTTP 400):
```json
{
  "data": null,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": [
      { "field": "fundId", "message": "Fund ID is required" },
      { "field": "fundName", "message": "Fund Name is required" }
    ]
  }
}
```

---

### 4. Submit UPDATE Change Request

```
PUT /accounts/requests/{requestId}
```

Submits an update to an existing account for approval (maker action).

**Path Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| requestId | long | The change request ID (for idempotency, or use POST with accountId) |

**Alternative**: `POST /accounts/requests` with `operationType: "UPDATE"` and `accountId` in body.

**Request Body**:
```json
{
  "operationType": "UPDATE",
  "accountId": "ABC123",
  "accountData": {
    "fundName": "AB Growth Equity Fund",
    "reportingFrequency": "Quarterly"
  }
}
```

**Response** (HTTP 200):
```json
{
  "data": {
    "requestId": 43,
    "operationType": "UPDATE",
    "accountId": "ABC123",
    "snapshot": {
      "fundId": "ABC123",
      "changes": {
        "fundName": { "from": "AB Equity Fund", "to": "AB Growth Equity Fund" },
        "reportingFrequency": { "from": "Monthly", "to": "Quarterly" }
      }
    },
    "makerLogin": "CORPORATE\\jdoe",
    "makerTimestamp": "2026-03-12T14:35:00Z",
    "status": "PENDING_APPROVAL"
  },
  "error": null
}
```

---

### 5. Submit DELETE Change Request

```
DELETE /accounts/requests/{requestId}
```

Submits a deactivation (soft-delete) for approval (maker action).

**Alternative**: `POST /accounts/requests` with `operationType: "DELETE"` and `accountId` in body.

**Request Body**:
```json
{
  "operationType": "DELETE",
  "accountId": "XYZ789"
}
```

**Response** (HTTP 200):
```json
{
  "data": {
    "requestId": 44,
    "operationType": "DELETE",
    "accountId": "XYZ789",
    "snapshot": {
      "fundId": "XYZ789",
      "fundName": "AB Legacy Fund",
      "fundType": "Legacy"
    },
    "makerLogin": "CORPORATE\\jdoe",
    "makerTimestamp": "2026-03-12T14:40:00Z",
    "status": "PENDING_APPROVAL"
  },
  "error": null
}
```

---

### 6. List Pending Change Requests (Checker Queue)

```
GET /accounts/requests
```

Returns all pending change requests for checker review.

**Query Parameters**:

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| status | string | No | "PENDING_APPROVAL" | Filter by status |
| makerLogin | string | No | null | Filter by maker (for "My Requests" view) |

**Response** (HTTP 200):
```json
{
  "data": [
    {
      "requestId": 42,
      "operationType": "CREATE",
      "accountId": null,
      "fundIdPreview": "DEF456",
      "fundNamePreview": "AB Fixed Income Fund",
      "makerLogin": "CORPORATE\\maker1",
      "makerTimestamp": "2026-03-12T14:30:00Z",
      "status": "PENDING_APPROVAL",
      "hasConflicts": false,
      "conflictCount": 0
    },
    {
      "requestId": 43,
      "operationType": "UPDATE",
      "accountId": "ABC123",
      "fundIdPreview": "ABC123",
      "fundNamePreview": "AB Growth Equity Fund",
      "makerLogin": "CORPORATE\\maker2",
      "makerTimestamp": "2026-03-12T14:35:00Z",
      "status": "PENDING_APPROVAL",
      "hasConflicts": true,
      "conflictCount": 1
    }
  ],
  "error": null
}
```

---

### 7. Get Change Request Details

```
GET /accounts/requests/{requestId}
```

Returns full details of a change request including snapshot.

**Response** (HTTP 200):
```json
{
  "data": {
    "requestId": 43,
    "operationType": "UPDATE",
    "accountId": "ABC123",
    "snapshot": {
      "fundId": "ABC123",
      "changes": {
        "fundName": { "from": "AB Equity Fund", "to": "AB Growth Equity Fund" }
      }
    },
    "currentAccount": {
      "fundId": "ABC123",
      "fundName": "AB Equity Fund",
      "fundType": "Equity"
    },
    "makerLogin": "CORPORATE\\maker2",
    "makerTimestamp": "2026-03-12T14:35:00Z",
    "status": "PENDING_APPROVAL",
    "conflictingRequests": [
      {
        "requestId": 45,
        "makerLogin": "CORPORATE\\maker3",
        "makerTimestamp": "2026-03-12T14:38:00Z"
      }
    ]
  },
  "error": null
}
```

---

### 8. Approve Change Request

```
POST /accounts/requests/{requestId}/approve
```

Approves a pending change request (checker action). Updates live Account atomically.

**Path Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| requestId | long | The change request ID |

**Request Body**: None (empty or `{}`)

**Response** (HTTP 200):
```json
{
  "data": {
    "requestId": 42,
    "status": "APPROVED",
    "checkerLogin": "CORPORATE\\checker1",
    "checkerTimestamp": "2026-03-12T15:00:00Z",
    "affectedAccount": {
      "fundId": "DEF456",
      "fundName": "AB Fixed Income Fund",
      "activeStatus": "ACTIVE"
    }
  },
  "error": null
}
```

**Error: Self-Approval Attempt** (HTTP 403):
```json
{
  "data": null,
  "error": {
    "code": "FORBIDDEN",
    "message": "You cannot approve your own request"
  }
}
```

**Error: Already Actioned** (HTTP 409):
```json
{
  "data": null,
  "error": {
    "code": "CONFLICT",
    "message": "Request has already been approved by another checker"
  }
}
```

---

### 9. Reject Change Request

```
POST /accounts/requests/{requestId}/reject
```

Rejects a pending change request with mandatory reason (checker action).

**Request Body**:
```json
{
  "reason": "Fund ID already exists in external ePace system. Please verify."
}
```

**Response** (HTTP 200):
```json
{
  "data": {
    "requestId": 43,
    "status": "REJECTED",
    "checkerLogin": "CORPORATE\\checker1",
    "checkerTimestamp": "2026-03-12T15:05:00Z",
    "rejectionReason": "Fund ID already exists in external ePace system. Please verify."
  },
  "error": null
}
```

**Error: Missing Reason** (HTTP 400):
```json
{
  "data": null,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Rejection reason is required"
  }
}
```

---

### 10. Cancel Own Pending Request

```
DELETE /accounts/requests/{requestId}/cancel
```

Cancels the maker's own pending request (maker action).

**Response** (HTTP 200):
```json
{
  "data": {
    "requestId": 44,
    "status": "CANCELLED"
  },
  "error": null
}
```

**Error: Not Own Request** (HTTP 403):
```json
{
  "data": null,
  "error": {
    "code": "FORBIDDEN",
    "message": "You can only cancel your own pending requests"
  }
}
```

**Error: Already Actioned** (HTTP 409):
```json
{
  "data": null,
  "error": {
    "code": "CONFLICT",
    "message": "Request has already been approved and cannot be cancelled"
  }
}
```

---

## Authentication Info Endpoint

```
GET /auth/me
```

Returns the authenticated user's Windows principal (for display in UI).

**Response** (HTTP 200):
```json
{
  "data": {
    "principal": "CORPORATE\\jdoe",
    "domain": "CORPORATE",
    "username": "jdoe"
  },
  "error": null
}
```

---

## Health Check

```
GET /actuator/health
```

Spring Boot Actuator health endpoint (unauthenticated for load balancer probes).

**Response** (HTTP 200):
```json
{
  "status": "UP",
  "components": {
    "db": { "status": "UP" },
    "diskSpace": { "status": "UP" }
  }
}
```
