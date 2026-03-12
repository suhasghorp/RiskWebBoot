# Implementation Plan: RiskWeb Core — Fund Account Management

**Branch**: `001-risk-web-core` | **Date**: 2026-03-12 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-riskweb-core/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/plan-template.md` for the execution workflow.

## Summary

Build the core Fund Account management system for RiskWeb — an AllianceBernstein-branded intranet application with Windows SSO (Kerberos/SPNEGO), maker-checker (4-eyes) approval workflow, and Syncfusion-based data grids. The application is a Spring Boot 4 + React 18 monolith deployed as a single fat JAR, connecting to an existing SQL Server database with no DDL migrations.

## Technical Context

**Language/Version**: Java 25 (backend), TypeScript with React 18+ (frontend)
**Primary Dependencies**: Spring Boot 4, Spring Security Kerberos, Spring Data JPA, Hibernate, MUI v6+, Syncfusion Grid/Forms/Charts, Axios, React Router v6+
**Storage**: Microsoft SQL Server (existing schema — no DDL migrations, HikariCP connection pool)
**Testing**: JUnit 5 with Mockito (backend mandatory), frontend testing not mandated in this phase
**Target Platform**: Windows intranet (Kerberos domain), deployed as fat JAR on Windows/Linux server
**Project Type**: Web application (Java backend + React frontend embedded in single JAR)
**Performance Goals**: <2 seconds for paginated account list with up to 500 accounts (SC-002)
**Constraints**: No containers (Docker/K8s prohibited), no DDL (`ddl-auto=validate` or `none`), stateless API sessions, maker ≠ checker enforcement at service layer
**Scale/Scope**: ~500 Fund Accounts, 10 user stories across 6 navigation sections (only Misc → Accounts functional)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Article | Requirement | Compliance Status |
|---------|-------------|-------------------|
| I.1 Backend | Java 25, Spring Boot 4, Maven, mssql-jdbc, Spring Data JPA, Spring Security Kerberos | ✅ COMPLIANT |
| I.2 Frontend | React 18+/TS, Vite, Axios, React Context, React Router v6+, MUI for shell, Syncfusion for grids/forms/charts | ✅ COMPLIANT |
| I.3 Database | SQL Server, no DDL migrations (Flyway/Liquibase prohibited) | ✅ COMPLIANT |
| I.4 Deployment | Single fat JAR with React embedded in `/src/main/resources/static` | ✅ COMPLIANT |
| II Authentication | Kerberos/SPNEGO SSO, no login page, Windows principal as identity | ✅ COMPLIANT |
| III Authorisation | Single ADMIN role, maker ≠ checker at service layer | ✅ COMPLIANT |
| IV Maker-Checker | All CRUD through AccountChangeRequest, atomic approval transactions | ✅ COMPLIANT |
| V Data Model | Account fields match spec, soft delete only, ddl-auto=validate/none | ✅ COMPLIANT |
| VI API Design | /api/v1/ prefix, REST conventions, consistent envelope, 403 for self-approve | ✅ COMPLIANT |
| VII Audit Trail | AccountChangeRequest immutable, UTC timestamps, no deletions | ✅ COMPLIANT |
| VIII Code Quality | Package structure per spec, DTOs only over API, SLF4J logging, Javadoc | ✅ COMPLIANT |
| IX Testing | JUnit 5, Mockito, mandatory maker-checker test coverage | ✅ COMPLIANT |
| X Brand | AB palette (#1E9BD7, #FFBF27, etc.), MUI theme, Syncfusion overrides, Inter font | ✅ COMPLIANT |
| XI Navigation | Three-zone layout, collapsible sidebar, navigation config file, WorkInProgress placeholder | ✅ COMPLIANT |

**Gate Status**: ✅ PASS — All constitution articles satisfied by design. No violations requiring justification.

## Project Structure

### Documentation (this feature)

```text
specs/001-riskweb-core/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (/speckit.plan command)
├── data-model.md        # Phase 1 output (/speckit.plan command)
├── quickstart.md        # Phase 1 output (/speckit.plan command)
├── contracts/           # Phase 1 output (/speckit.plan command)
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (repository root)

```text
# Backend (Java/Spring Boot) — per Constitution Article VIII.1
src/main/java/com/ab/riskweb/
├── config/              # Security (Kerberos), JPA, Web MVC config
├── domain/              # JPA entities (Account, AccountChangeRequest)
├── repository/          # Spring Data JPA repositories
├── service/             # Business logic (maker-checker enforcement)
├── controller/          # REST controllers (thin — delegate to service)
├── dto/                 # Request/response DTOs (no entity exposure over API)
└── exception/           # Custom exceptions and @ControllerAdvice handler

src/main/resources/
├── application.yml      # Spring Boot config (datasource, security, etc.)
└── static/              # React build output (embedded at build time)

src/test/java/com/ab/riskweb/
├── service/             # Service unit tests (mandatory maker-checker tests)
├── controller/          # Controller tests (optional)
└── repository/          # Repository tests (optional)

# Frontend (React/TypeScript) — per Constitution Article VIII.2
frontend/
├── src/
│   ├── api/             # Axios client + typed API functions
│   ├── assets/          # Logo files (logo-1.png, logo-2.png)
│   ├── components/      # Reusable MUI-based components
│   ├── config/          # navigation.ts config file
│   ├── context/         # React Context providers (auth, notifications)
│   ├── hooks/           # Custom React hooks
│   ├── pages/           # Route-level page components
│   ├── theme/           # abTheme.ts, syncfusionOverrides.css
│   └── types/           # TypeScript interfaces mirroring backend DTOs
├── index.html
├── vite.config.ts
├── tsconfig.json
└── package.json

# Build output integration
pom.xml                  # Maven build with frontend-maven-plugin
```

**Structure Decision**: Embedded monolith pattern per Constitution Article I.4. The React frontend is built via `frontend-maven-plugin` and copied to `src/main/resources/static/` at build time, producing a single deployable fat JAR. No separate frontend deployment or containers.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

### Post-Constitution Clarifications (Not Violations)

During detailed design, the following requirements were added to clarify Article IV (Maker-Checker Workflow) behavior for edge cases not explicitly addressed in the constitution:

**FR-014a & FR-014b: Concurrent UPDATE Request Handling**
- **Rationale**: Constitution Article IV describes the maker-checker workflow but is silent on how to handle multiple simultaneous UPDATE requests for the same account. This is a clarification of existing Article IV requirements, not a deviation.
- **Approach**:
  - Last-write-wins conflict resolution (FR-014b): When multiple UPDATE requests are approved sequentially, the later approval overwrites earlier changes to the same field. This aligns with standard database semantics and Article IV's requirement for atomic transaction updates.
  - Conflict warning UI (FR-014a): The checker UI displays a warning badge when reviewing an UPDATE request if other PENDING_APPROVAL UPDATE requests exist for the same account. This is a defensive UX measure to support informed decision-making by checkers.
- **Constitutional Authority**: These clarifications extend Article IV's mandate for maker-checker workflow integrity without contradicting any existing article. No constitutional amendment required.
- **Impact**: Added tasks T079 (warning badge implementation), updated service layer logic to detect conflicts (T070), no additional complexity beyond edge case handling.

**HikariCP Connection Pool Sizing (FR-015b clarification)**
- **Rationale**: See research.md "Database Connection Pooling" section for detailed justification of pool size parameters (maximum-pool-size: 10, minimum-idle: 5).

All other design decisions align with constitution articles without requiring justification.
