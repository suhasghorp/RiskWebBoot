# Data Model: RiskWeb Core — Fund Account Management

**Branch**: `001-risk-web-core` | **Date**: 2026-03-12

This document defines the entity structures, relationships, and validation rules extracted from the feature specification and constitution.

---

## Entity Relationship Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              ACCOUNT                                     │
│  (Live Fund Account record — soft delete only)                          │
├─────────────────────────────────────────────────────────────────────────┤
│  fund_id          VARCHAR(50)   PK   NOT NULL  -- Business key          │
│  fund_name        VARCHAR(255)       NOT NULL  -- Required              │
│  epace_id         VARCHAR(100)       NULL      -- Free text             │
│  short_name       VARCHAR(100)       NULL                               │
│  fund_type        VARCHAR(50)        NOT NULL  -- Required              │
│  reporting_freq   VARCHAR(50)        NULL                               │
│  esma_40act       VARCHAR(50)        NULL                               │
│  rm_acronym       VARCHAR(50)        NULL      -- Free text             │
│  active_status    VARCHAR(20)        NOT NULL  -- 'ACTIVE'/'INACTIVE'   │
│  created_by       VARCHAR(100)       NOT NULL  -- Windows login         │
│  created_on       DATETIME2          NOT NULL  -- UTC timestamp         │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ 1:N (account_id nullable for CREATE)
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                       ACCOUNT_CHANGE_REQUEST                             │
│  (Maker-checker workflow record — immutable audit trail)                │
├─────────────────────────────────────────────────────────────────────────┤
│  request_id       BIGINT        PK   IDENTITY  -- Auto-generated        │
│  operation_type   VARCHAR(20)        NOT NULL  -- CREATE/UPDATE/DELETE  │
│  account_id       VARCHAR(50)        NULL      -- FK, null for CREATE   │
│  snapshot         NVARCHAR(MAX)      NOT NULL  -- JSON of proposed vals │
│  maker_login      VARCHAR(100)       NOT NULL  -- Windows principal     │
│  maker_timestamp  DATETIME2          NOT NULL  -- UTC                   │
│  checker_login    VARCHAR(100)       NULL      -- Set on approve/reject │
│  checker_timestamp DATETIME2         NULL      -- Set on approve/reject │
│  status           VARCHAR(20)        NOT NULL  -- Enum: see below       │
│  rejection_reason NVARCHAR(1000)     NULL      -- Required if REJECTED  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Entity: Account

### JPA Mapping

```java
@Entity
@Table(name = "account")
public class Account {

    @Id
    @Column(name = "fund_id", length = 50, nullable = false)
    private String fundId;

    @Column(name = "fund_name", length = 255, nullable = false)
    private String fundName;

    @Column(name = "epace_id", length = 100)
    private String epaceId;

    @Column(name = "short_name", length = 100)
    private String shortName;

    @Column(name = "fund_type", length = 50, nullable = false)
    private String fundType;

    @Column(name = "reporting_freq", length = 50)
    private String reportingFrequency;

    @Column(name = "esma_40act", length = 50)
    private String esma40Act;

    @Column(name = "rm_acronym", length = 50)
    private String rmAcronym;

    @Enumerated(EnumType.STRING)
    @Column(name = "active_status", length = 20, nullable = false)
    private ActiveStatus activeStatus = ActiveStatus.ACTIVE;

    @Column(name = "created_by", length = 100, nullable = false)
    private String createdBy;

    @Column(name = "created_on", nullable = false)
    private Instant createdOn;

    // Getters/setters omitted
}
```

### Enums

```java
public enum ActiveStatus {
    ACTIVE,
    INACTIVE
}
```

### Validation Rules

| Field | Rule | Error Message |
|-------|------|---------------|
| fundId | Required, max 50 chars | "Fund ID is required" |
| fundName | Required, max 255 chars | "Fund Name is required" |
| fundType | Required, max 50 chars | "Fund Type is required" |
| activeStatus | Default ACTIVE, set by system | N/A (not user-editable) |
| createdBy | Set by system on CREATE approval | N/A (not user-editable) |
| createdOn | Set by system on CREATE approval | N/A (not user-editable) |

### Query Defaults

- All list queries filter by `activeStatus = ACTIVE` unless explicitly requesting inactive
- Soft delete sets `activeStatus = INACTIVE` (no hard deletes)

---

## Entity: AccountChangeRequest

### JPA Mapping

```java
@Entity
@Table(name = "account_change_request")
public class AccountChangeRequest {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "request_id")
    private Long requestId;

    @Enumerated(EnumType.STRING)
    @Column(name = "operation_type", length = 20, nullable = false)
    private OperationType operationType;

    @Column(name = "account_id", length = 50)
    private String accountId;  // null for CREATE, populated for UPDATE/DELETE

    @Column(name = "snapshot", columnDefinition = "NVARCHAR(MAX)", nullable = false)
    @Convert(converter = JsonNodeConverter.class)
    private JsonNode snapshot;

    @Column(name = "maker_login", length = 100, nullable = false)
    private String makerLogin;

    @Column(name = "maker_timestamp", nullable = false)
    private Instant makerTimestamp;

    @Column(name = "checker_login", length = 100)
    private String checkerLogin;

    @Column(name = "checker_timestamp")
    private Instant checkerTimestamp;

    @Enumerated(EnumType.STRING)
    @Column(name = "status", length = 20, nullable = false)
    private ChangeRequestStatus status = ChangeRequestStatus.PENDING_APPROVAL;

    @Column(name = "rejection_reason", length = 1000)
    private String rejectionReason;

    // Getters/setters omitted
}
```

### Enums

```java
public enum OperationType {
    CREATE,
    UPDATE,
    DELETE
}

public enum ChangeRequestStatus {
    PENDING_APPROVAL,
    APPROVED,
    REJECTED,
    CANCELLED;

    /**
     * Validates state transitions. Only PENDING_APPROVAL can transition to other states.
     */
    public boolean canTransitionTo(ChangeRequestStatus target) {
        if (this != PENDING_APPROVAL) {
            return false;
        }
        return target == APPROVED || target == REJECTED || target == CANCELLED;
    }
}
```

### State Transitions

| From | To | Trigger | Actor |
|------|-----|---------|-------|
| PENDING_APPROVAL | APPROVED | Checker approves | Checker (≠ maker) |
| PENDING_APPROVAL | REJECTED | Checker rejects with reason | Checker (≠ maker) |
| PENDING_APPROVAL | CANCELLED | Maker cancels own request | Original maker only |

### Validation Rules

| Field | Rule | Error Message |
|-------|------|---------------|
| operationType | Required, enum value | "Operation type is required" |
| accountId | Required for UPDATE/DELETE, null for CREATE | "Account ID required for updates" |
| snapshot | Required, valid JSON | "Snapshot is required" |
| makerLogin | Required, Windows principal format | "Maker login is required" |
| rejectionReason | Required if status = REJECTED | "Rejection reason is required" |

### Audit Trail Constraints (Constitution Article VII)

- Records MUST NEVER be deleted (no DELETE statements)
- All timestamps stored in UTC
- Immutable after creation (updates only to checker fields and status)

---

## JSON Snapshot Schema

The `snapshot` field stores proposed field values for checker review:

### CREATE Operation Snapshot

```json
{
  "fundId": "ABC123",
  "fundName": "AB Equity Fund",
  "epaceId": "EP-001",
  "shortName": "Equity",
  "fundType": "Equity",
  "reportingFrequency": "Monthly",
  "esma40Act": "ESMA",
  "rmAcronym": "EQ"
}
```

### UPDATE Operation Snapshot

```json
{
  "fundId": "ABC123",
  "changes": {
    "fundName": {
      "from": "AB Equity Fund",
      "to": "AB Growth Equity Fund"
    },
    "reportingFrequency": {
      "from": "Monthly",
      "to": "Quarterly"
    }
  }
}
```

### DELETE Operation Snapshot

```json
{
  "fundId": "ABC123",
  "fundName": "AB Equity Fund",
  "fundType": "Equity",
  "reason": "Fund liquidated"
}
```

---

## JSON Snapshot Format Specification

The `snapshot` field in `AccountChangeRequest` stores the proposed account data as a JSON object. The format varies by operation type:

### CREATE Operation

The snapshot contains all submitted form fields, including:
- **Required fields** (fundId, fundName, fundType): Always present with non-null values
- **Optional fields** (epaceId, shortName, reportingFrequency, esma40Act, rmAcronym):
  - **Included if user provided a value** (even if empty string)
  - **Omitted if user left field blank/untouched** (null values not serialized)

**Example CREATE snapshot**:
```json
{
  "fundId": "ABC123",
  "fundName": "Test Fund",
  "fundType": "EQUITY",
  "shortName": "TestF",
  "reportingFrequency": "MONTHLY"
  // epaceId, esma40Act, rmAcronym omitted (user left blank)
}
```

**Rationale**: Omitting null values reduces snapshot size and makes checker review clearer (only shows what the maker explicitly provided).

### UPDATE Operation

The snapshot uses a **diff format** containing only changed fields with before/after values:

**Example UPDATE snapshot**:
```json
{
  "fundName": {
    "from": "Old Fund Name",
    "to": "New Fund Name"
  },
  "reportingFrequency": {
    "from": "QUARTERLY",
    "to": "MONTHLY"
  }
}
```

**Rationale**: The diff format enables the checker UI to display side-by-side comparisons (current vs. proposed) and supports audit trail analysis of what changed.

### DELETE Operation

The snapshot contains the **complete current state** of the account being deactivated:

**Example DELETE snapshot**:
```json
{
  "fundId": "XYZ789",
  "fundName": "Archived Fund",
  "fundType": "FIXED_INCOME",
  "activeStatus": "ACTIVE"
  // ... all other fields
}
```

**Rationale**: Preserving the full account state at deactivation time provides an audit record of what was active before soft-delete.

---

## TypeScript Interfaces (Frontend)

```typescript
// types/account.ts
export interface Account {
  fundId: string;
  fundName: string;
  epaceId?: string;
  shortName?: string;
  fundType: string;
  reportingFrequency?: string;
  esma40Act?: string;
  rmAcronym?: string;
  activeStatus: 'ACTIVE' | 'INACTIVE';
  createdBy: string;
  createdOn: string; // ISO 8601
}

export interface AccountChangeRequest {
  requestId: number;
  operationType: 'CREATE' | 'UPDATE' | 'DELETE';
  accountId?: string;
  snapshot: Record<string, unknown>;
  makerLogin: string;
  makerTimestamp: string; // ISO 8601
  checkerLogin?: string;
  checkerTimestamp?: string; // ISO 8601
  status: 'PENDING_APPROVAL' | 'APPROVED' | 'REJECTED' | 'CANCELLED';
  rejectionReason?: string;
}

// Form input types (subset for validation)
export interface AccountFormInput {
  fundId: string;        // Required
  fundName: string;      // Required
  epaceId?: string;
  shortName?: string;
  fundType: string;      // Required
  reportingFrequency?: string;
  esma40Act?: string;
  rmAcronym?: string;
}
```

---

## Repository Interfaces

```java
// AccountRepository.java
@Repository
public interface AccountRepository extends JpaRepository<Account, String> {

    /**
     * Find all active accounts (default view).
     */
    List<Account> findByActiveStatus(ActiveStatus status);

    /**
     * Paginated active accounts with search.
     */
    @Query("SELECT a FROM Account a WHERE a.activeStatus = :status " +
           "AND (LOWER(a.fundName) LIKE LOWER(CONCAT('%', :search, '%')) " +
           "OR LOWER(a.fundType) LIKE LOWER(CONCAT('%', :search, '%')) " +
           "OR LOWER(a.shortName) LIKE LOWER(CONCAT('%', :search, '%')))")
    Page<Account> searchActive(@Param("status") ActiveStatus status,
                               @Param("search") String search,
                               Pageable pageable);
}

// AccountChangeRequestRepository.java
@Repository
public interface AccountChangeRequestRepository
    extends JpaRepository<AccountChangeRequest, Long> {

    /**
     * Find all pending requests for checker queue.
     */
    List<AccountChangeRequest> findByStatus(ChangeRequestStatus status);

    /**
     * Find pending requests by maker (for "My Pending Requests" view).
     */
    List<AccountChangeRequest> findByMakerLoginAndStatus(String makerLogin,
                                                          ChangeRequestStatus status);

    /**
     * Find conflicting pending UPDATE requests for the same account.
     */
    List<AccountChangeRequest> findByAccountIdAndStatusAndRequestIdNot(
        String accountId,
        ChangeRequestStatus status,
        Long excludeRequestId
    );
}
```

---

## Database Constraints

Per Constitution Article V.3, the application does NOT create or modify tables. The DBA team owns the schema. Expected constraints on the existing tables:

### account table
- PRIMARY KEY on `fund_id`
- NOT NULL on `fund_id`, `fund_name`, `fund_type`, `active_status`, `created_by`, `created_on`

### account_change_request table
- PRIMARY KEY on `request_id` (IDENTITY)
- NOT NULL on `operation_type`, `snapshot`, `maker_login`, `maker_timestamp`, `status`
- CHECK constraint on `status` values
- CHECK constraint on `operation_type` values
- FK on `account_id` to `account(fund_id)` (nullable, CASCADE on UPDATE, NO ACTION on DELETE)
