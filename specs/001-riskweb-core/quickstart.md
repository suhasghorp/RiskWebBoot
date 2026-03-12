# Quickstart: RiskWeb Core

**Branch**: `001-risk-web-core` | **Date**: 2026-03-12

This guide covers environment setup, build commands, and local development workflow for RiskWeb.

---

## Prerequisites

### Required Software

| Software | Version | Purpose |
|----------|---------|---------|
| Java JDK | 25+ | Backend runtime |
| Maven | 3.9+ | Build tool |
| Node.js | 20.x LTS | Frontend tooling |
| npm | 10.x | Package manager |
| Git | 2.x | Version control |

### Development Environment

- **IDE**: IntelliJ IDEA (recommended) or VS Code
- **Database**: Access to SQL Server instance with RiskWeb schema
- **Kerberos**: For local SSO testing, a keytab file or MIT Kerberos Ticket Manager

---

## Environment Variables

Create a `.env` file at the repository root (gitignored):

```bash
# Database
DB_HOST=sqlserver.internal
DB_PORT=1433
DB_NAME=RiskWeb
DB_USERNAME=riskweb_app
DB_PASSWORD=<secure-password>

# Kerberos (for local development, may use fallback auth)
KERBEROS_PRINCIPAL=HTTP/riskweb.internal@CORPORATE.LOCAL
KERBEROS_KEYTAB=/etc/riskweb/riskweb.keytab

# Syncfusion License (required)
VITE_SYNCFUSION_LICENSE_KEY=<your-license-key>
```

---

## Project Structure

```
RiskWeb/
├── pom.xml                          # Maven parent POM
├── src/
│   ├── main/
│   │   ├── java/com/ab/riskweb/     # Java source
│   │   └── resources/
│   │       ├── application.yml      # Spring Boot config
│   │       └── static/              # React build output (generated)
│   └── test/java/                   # JUnit tests
├── frontend/                        # React application
│   ├── src/
│   ├── package.json
│   ├── vite.config.ts
│   └── tsconfig.json
└── specs/                           # Feature specifications
```

---

## Build Commands

### Full Build (Production JAR)

```bash
# Build everything: frontend + backend + fat JAR
mvn clean package

# Output: target/riskweb-1.0.0.jar
```

### Backend Only

```bash
# Compile backend without frontend build
mvn compile -DskipFrontend

# Run tests
mvn test

# Run with Spring Boot devtools
mvn spring-boot:run
```

### Frontend Only

```bash
# Navigate to frontend directory
cd frontend

# Install dependencies
npm install

# Development server with hot reload (proxies to backend)
npm run dev

# Production build
npm run build

# Type checking
npm run typecheck

# Linting
npm run lint
```

---

## Local Development Workflow

### Option 1: Full Stack (Recommended)

Run both backend and frontend with hot reload:

**Terminal 1 - Backend**:
```bash
mvn spring-boot:run -Dspring-boot.run.profiles=local
```

**Terminal 2 - Frontend**:
```bash
cd frontend && npm run dev
```

Frontend dev server runs on `http://localhost:5173` and proxies `/api/*` to `http://localhost:8080`.

### Option 2: Backend Only (API Development)

```bash
# Use embedded H2 for faster iteration (if configured)
mvn spring-boot:run -Dspring-boot.run.profiles=local,h2

# Or connect to real SQL Server
mvn spring-boot:run -Dspring-boot.run.profiles=local
```

Test API with curl or Postman (authentication bypassed in local profile).

### Option 3: Frontend Only (UI Development)

```bash
cd frontend
npm run dev

# Uses mock API responses (if configured in vite.config.ts)
```

---

## Configuration Profiles

### application.yml

```yaml
spring:
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:local}

---
# Local development profile
spring:
  config:
    activate:
      on-profile: local
  datasource:
    url: jdbc:sqlserver://${DB_HOST:localhost}:${DB_PORT:1433};databaseName=${DB_NAME:RiskWeb};encrypt=false
    username: ${DB_USERNAME:sa}
    password: ${DB_PASSWORD:password}
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: true
  security:
    # Bypass Kerberos for local development
    kerberos:
      enabled: false

# Mock user for local development (when Kerberos disabled)
riskweb:
  mock-user:
    enabled: true
    principal: LOCAL\\developer

---
# Production profile
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: jdbc:sqlserver://${DB_HOST}:${DB_PORT};databaseName=${DB_NAME};encrypt=true;trustServerCertificate=false
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: none
    show-sql: false
  security:
    kerberos:
      enabled: true
      service-principal: ${KERBEROS_PRINCIPAL}
      keytab-location: file:${KERBEROS_KEYTAB}
```

---

## Testing

### Run All Tests

```bash
mvn test
```

### Run Specific Test Class

```bash
mvn test -Dtest=AccountChangeRequestServiceTest
```

### Run with Coverage Report

```bash
mvn test jacoco:report
# Report: target/site/jacoco/index.html
```

### Critical Test: Maker-Checker Enforcement

The following test MUST pass. If removed or broken, the build should fail:

```java
@Test
void approveOwnRequest_shouldThrowForbidden() {
    // Given: A pending request created by "CORPORATE\jdoe"
    AccountChangeRequest request = createPendingRequest("CORPORATE\\jdoe");

    // When: The same user attempts to approve it
    // Then: ForbiddenException is thrown
    assertThrows(ForbiddenException.class, () ->
        service.approve(request.getRequestId(), "CORPORATE\\jdoe")
    );
}
```

---

## Database Setup

### Schema Verification

The application does NOT create tables. Verify the schema exists:

```sql
-- Check account table
SELECT COLUMN_NAME, DATA_TYPE, IS_NULLABLE
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_NAME = 'account';

-- Check change request table
SELECT COLUMN_NAME, DATA_TYPE, IS_NULLABLE
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_NAME = 'account_change_request';
```

### Sample Data (Development)

```sql
-- Insert test accounts
INSERT INTO account (fund_id, fund_name, fund_type, active_status, created_by, created_on)
VALUES
  ('TEST001', 'Test Equity Fund', 'Equity', 'ACTIVE', 'LOCAL\developer', GETUTCDATE()),
  ('TEST002', 'Test Fixed Income Fund', 'Fixed Income', 'ACTIVE', 'LOCAL\developer', GETUTCDATE());
```

---

## IDE Setup

### IntelliJ IDEA

1. Open project as Maven project
2. Enable annotation processing: Settings → Build → Compiler → Annotation Processors
3. Install Lombok plugin (if using Lombok)
4. Configure Node.js: Settings → Languages & Frameworks → Node.js
5. Run configuration:
   - Spring Boot: `RiskWebApplication` with profile `local`
   - npm: Working directory `frontend`, command `run dev`

### VS Code

1. Install extensions:
   - Extension Pack for Java
   - Spring Boot Extension Pack
   - ES7+ React/Redux/React-Native snippets
   - Prettier
2. Open workspace: `File → Open Folder → RiskWeb`
3. Frontend terminal: `cd frontend && npm run dev`
4. Backend: Use Spring Boot Dashboard or Maven tasks

---

## Deployment

### Build Production JAR

```bash
# Clean build with all tests
mvn clean package -Pprod

# Skip tests (not recommended)
mvn clean package -DskipTests
```

### Run Production JAR

```bash
java -jar target/riskweb-1.0.0.jar \
  --spring.profiles.active=prod \
  --DB_HOST=sqlserver.prod.internal \
  --DB_USERNAME=riskweb_prod \
  --DB_PASSWORD=<secure> \
  --KERBEROS_KEYTAB=/etc/riskweb/riskweb.keytab
```

### Health Check

```bash
curl http://localhost:8080/actuator/health
```

---

## Troubleshooting

### Common Issues

| Issue | Solution |
|-------|----------|
| `Syncfusion license warning` | Set `VITE_SYNCFUSION_LICENSE_KEY` env var |
| `401 Unauthorized` locally | Enable `riskweb.mock-user.enabled=true` |
| `Connection refused` to DB | Check DB_HOST, DB_PORT, firewall rules |
| `ddl-auto validation failed` | Schema mismatch — contact DBA |
| Frontend proxy error | Ensure backend is running on port 8080 |

### Logs

```bash
# Spring Boot logs
tail -f logs/riskweb.log

# Enable debug logging
java -jar riskweb.jar --logging.level.com.ab.riskweb=DEBUG
```

---

## Next Steps

1. **Run `/speckit.tasks`** to generate implementation tasks
2. **Start with User Story 10** (Navigation Shell) — foundation for all other stories
3. **Then User Story 1** (SSO Authentication) — required for all API calls
4. **Then User Story 2** (Account List) — first data-driven feature
