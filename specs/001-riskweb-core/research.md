# Research: RiskWeb Core — Fund Account Management

**Branch**: `001-risk-web-core` | **Date**: 2026-03-12

This document consolidates research findings for all technology choices and potential unknowns identified during planning.

---

## 1. Spring Security Kerberos/SPNEGO Configuration

### Decision
Use `spring-security-kerberos` extension with `SpnegoAuthenticationProcessingFilter` and `KerberosServiceAuthenticationProvider`.

### Rationale
- Spring Security Kerberos is the official Spring project for Kerberos/SPNEGO authentication
- Integrates seamlessly with Spring Boot 4 auto-configuration
- Supports stateless ticket validation via `SunJaasKerberosTicketValidator`
- Principal extraction provides `DOMAIN\username` format required for audit trails

### Alternatives Considered
| Alternative | Rejected Because |
|-------------|------------------|
| Manual JAAS configuration | More boilerplate, less Spring integration |
| Apache HTTP Server with mod_auth_kerb | Adds infrastructure dependency, violates single-JAR deployment |
| WAFFLE (Windows-only) | Not cross-platform for development/testing |

### Configuration Requirements
```yaml
# application.yml
spring:
  security:
    kerberos:
      service-principal: HTTP/riskweb.internal@CORPORATE.LOCAL
      keytab-location: file:/etc/riskweb/riskweb.keytab
```

### Key Classes
- `SpnegoAuthenticationProcessingFilter` — intercepts WWW-Authenticate: Negotiate
- `KerberosServiceAuthenticationProvider` — validates Kerberos ticket
- `SunJaasKerberosTicketValidator` — uses JAAS to validate tickets server-side

---

## 2. Syncfusion License Key Handling

### Decision
Register Syncfusion license via `registerLicense()` at React app entry point, sourced from `VITE_SYNCFUSION_LICENSE_KEY` environment variable.

### Rationale
- Syncfusion requires license registration before any component mounts
- Environment variable approach keeps secrets out of source control
- Vite exposes `VITE_*` prefixed env vars to client-side code safely

### Implementation
```typescript
// main.tsx
import { registerLicense } from '@syncfusion/ej2-base';

const licenseKey = import.meta.env.VITE_SYNCFUSION_LICENSE_KEY;
if (!licenseKey) {
  console.error('FATAL: VITE_SYNCFUSION_LICENSE_KEY not set');
  throw new Error('Syncfusion license key required');
}
registerLicense(licenseKey);
```

### Alternatives Considered
| Alternative | Rejected Because |
|-------------|------------------|
| Hardcoded license key | Security risk, constitution violation |
| Backend-served license | Unnecessary complexity, license needed at mount time |
| Lazy registration | Components render before registration, causing warnings |

---

## 3. Maven Frontend Plugin Integration

### Decision
Use `frontend-maven-plugin` to build React app and copy output to `src/main/resources/static/`.

### Rationale
- Single build command (`mvn package`) produces complete fat JAR
- No separate frontend deployment pipeline needed
- React build output served by Spring Boot's static resource handling
- SPA routing handled by Spring MVC fallback controller

### Configuration
```xml
<!-- pom.xml -->
<plugin>
  <groupId>com.github.eirslett</groupId>
  <artifactId>frontend-maven-plugin</artifactId>
  <version>1.15.0</version>
  <configuration>
    <workingDirectory>frontend</workingDirectory>
    <nodeVersion>v20.11.0</nodeVersion>
  </configuration>
  <executions>
    <execution>
      <id>install-node-npm</id>
      <goals><goal>install-node-and-npm</goal></goals>
    </execution>
    <execution>
      <id>npm-install</id>
      <goals><goal>npm</goal></goals>
      <configuration><arguments>install</arguments></configuration>
    </execution>
    <execution>
      <id>npm-build</id>
      <goals><goal>npm</goal></goals>
      <configuration><arguments>run build</arguments></configuration>
    </execution>
  </executions>
</plugin>

<plugin>
  <artifactId>maven-resources-plugin</artifactId>
  <executions>
    <execution>
      <id>copy-frontend</id>
      <phase>prepare-package</phase>
      <goals><goal>copy-resources</goal></goals>
      <configuration>
        <outputDirectory>${project.build.directory}/classes/static</outputDirectory>
        <resources>
          <resource><directory>frontend/dist</directory></resource>
        </resources>
      </configuration>
    </execution>
  </executions>
</plugin>
```

### SPA Fallback Controller
```java
@Controller
public class SpaFallbackController {
    @RequestMapping(value = "/{path:[^\\.]*}")
    public String forward() {
        return "forward:/index.html";
    }
}
```

---

## 4. HikariCP SQL Server Configuration

### Decision
Use HikariCP (Spring Boot default) with SQL Server-optimized settings.

### Rationale
- HikariCP is Spring Boot's default connection pool (transitive via `spring-boot-starter-data-jpa`)
- Fastest Java connection pool with minimal overhead
- SQL Server driver (`mssql-jdbc`) has full HikariCP compatibility

### Configuration
```yaml
# application.yml
spring:
  datasource:
    url: jdbc:sqlserver://sqlserver.internal:1433;databaseName=RiskWeb;encrypt=true;trustServerCertificate=true
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    driver-class-name: com.microsoft.sqlserver.jdbc.SQLServerDriver
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
      idle-timeout: 300000
      connection-timeout: 20000
      max-lifetime: 1200000
```

### Alternatives Considered
| Alternative | Rejected Because |
|-------------|------------------|
| Apache DBCP2 | Slower, more configuration |
| C3P0 | Legacy, less maintained |
| Tomcat JDBC | Reasonable alternative but HikariCP is faster |

---

## 5. Maker-Checker State Machine

### Decision
Implement status transitions as enum with explicit allowed transitions.

### Rationale
- Clear state machine prevents invalid status changes
- Service layer enforces transitions before persistence
- Testable in isolation with unit tests

### State Diagram
```
                      ┌──────────────┐
                      │   PENDING    │
                      │   APPROVAL   │
                      └──────────────┘
                            │
           ┌────────────────┼────────────────┐
           ▼                ▼                ▼
    ┌──────────┐     ┌──────────┐     ┌──────────┐
    │ APPROVED │     │ REJECTED │     │CANCELLED │
    └──────────┘     └──────────┘     └──────────┘
         │
         ▼
    [Live Account Updated]
```

### Enum Definition
```java
public enum ChangeRequestStatus {
    PENDING_APPROVAL,
    APPROVED,
    REJECTED,
    CANCELLED;

    public boolean canTransitionTo(ChangeRequestStatus target) {
        if (this != PENDING_APPROVAL) return false;
        return target == APPROVED || target == REJECTED || target == CANCELLED;
    }
}
```

---

## 6. JSON Snapshot Storage

### Decision
Store proposed field values as JSON in `AccountChangeRequest.snapshot` column using Jackson.

### Rationale
- JSON preserves all field values without schema changes for new fields
- Allows diff display in checker UI (current vs proposed)
- Immutable snapshot for audit trail
- Jackson is already included via Spring Boot

### Implementation
```java
@Entity
public class AccountChangeRequest {
    @Column(columnDefinition = "NVARCHAR(MAX)")
    @Convert(converter = JsonNodeConverter.class)
    private JsonNode snapshot;
}

@Converter
public class JsonNodeConverter implements AttributeConverter<JsonNode, String> {
    private static final ObjectMapper mapper = new ObjectMapper();

    @Override
    public String convertToDatabaseColumn(JsonNode node) {
        return node == null ? null : node.toString();
    }

    @Override
    public JsonNode convertToEntityAttribute(String json) {
        try {
            return json == null ? null : mapper.readTree(json);
        } catch (JsonProcessingException e) {
            throw new IllegalArgumentException("Invalid JSON snapshot", e);
        }
    }
}
```

---

## 7. Conflict Detection for Concurrent Updates

### Decision
Implement warning badge in checker UI when multiple pending UPDATE requests exist for the same account.

### Rationale
- Last-write-wins is acceptable per spec (FR-014b)
- Warning empowers checker to review carefully
- No optimistic locking needed at request level

### Implementation
```java
// Service method
public List<AccountChangeRequest> findConflictingRequests(Long accountId, Long excludeRequestId) {
    return repository.findByAccountIdAndStatusAndIdNot(
        accountId,
        ChangeRequestStatus.PENDING_APPROVAL,
        excludeRequestId
    );
}
```

```typescript
// Checker UI
{conflictingRequests.length > 0 && (
  <WarningBadge>
    Warning: {conflictingRequests.length} other pending request(s) exist for this account.
  </WarningBadge>
)}
```

---

## Summary of Resolved Items

| Item | Resolution |
|------|------------|
| Kerberos/SPNEGO setup | spring-security-kerberos with SunJaasKerberosTicketValidator |
| Syncfusion licensing | VITE_SYNCFUSION_LICENSE_KEY env var, fail-fast on missing |
| Frontend build integration | frontend-maven-plugin with copy-resources to static/ |
| Connection pooling | HikariCP (Spring Boot default) with SQL Server optimizations |
| Workflow state machine | Enum-based with explicit transition validation |
| Field snapshot storage | JSON column with Jackson converter |
| Concurrent edit conflicts | Warning badge, last-write-wins resolution |

All NEEDS CLARIFICATION items have been resolved. Ready for Phase 1 design artifacts.

---

## 8. Database Connection Pooling

### HikariCP Configuration Rationale

**Selected Pool Sizing** (per tasks.md T176):
- `maximum-pool-size: 10`
- `minimum-idle: 5`

### Justification

#### 1. Expected Concurrency
Based on the application scope (Fund Account management for a single business unit), we estimate:
- ~10-20 concurrent ADMIN users during peak hours
- Typical session pattern: Browse accounts (1 query) → Submit change request (1 write) → idle
- Average transaction duration: <100ms (single-table CRUD operations)
- Peak concurrent transactions: ~5-8 (estimated)

#### 2. Pool Size Calculation
- **Maximum pool size (10)**: Set to ~2× peak concurrent transactions to handle burst traffic (e.g., end-of-month reconciliation periods when multiple users approve pending requests simultaneously)
- **Minimum idle (5)**: Matches average concurrent transaction estimate, avoiding connection churn during normal operation while keeping resource usage reasonable

#### 3. SQL Server Constraints
- Default SQL Server max connections: 32,767 (unlimited for practical purposes)
- No organizational limits documented; 10 connections represents <0.1% of server capacity
- Intranet latency: <5ms typical; connection acquisition overhead negligible

#### 4. Tuning Strategy
Start with these conservative defaults and monitor:
- Monitor HikariCP metrics via Spring Boot Actuator: `hikaricp.connections.active`, `hikaricp.connections.pending`, `hikaricp.connections.timeout`
- Increase `maximum-pool-size` if `connections.pending` exceeds 0 during normal operation
- Decrease `minimum-idle` if `connections.active` consistently stays below 3

### Alternative Considered
- **Spring Boot default**: `maximum-pool-size: 10` (coincidentally same), `minimum-idle: 10`
- **Rejected**: Setting minimum-idle = maximum defeats the purpose of a dynamic pool; keeps 10 connections alive even during off-peak hours (wasteful)

### References
- HikariCP sizing guide: https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing
- Formula: `connections = ((core_count * 2) + effective_spindle_count)` → For I/O-bound workloads, start conservative and tune empirically
