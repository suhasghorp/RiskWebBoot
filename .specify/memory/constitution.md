# Project Constitution

<!-- Sync Impact Report
Version: 1.2.0
Ratified: 2026-03-12
Last Amended: 2026-03-12
Principles Count: 11
Placeholders Resolved: All
Pending TODOs: None
Amendment Notes:
  - Article I.2 updated: MUI retained for shell only; Syncfusion added for Grid/Forms/Charts
  - Article X added: AllianceBernstein brand guidelines (logos, palette, MUI theme, Syncfusion theme)
  - Article XI added: Application layout, navigation structure, shell behaviour
  - Governance updated: Articles I-XI (was I-X)
  - X.2 usage rules updated: main content area background changed from White to Black (#000000)
  - Project name updated: RiskWeb
-->

**Project:** RiskWeb — Fund Account Management System  
**Constitution Version:** 1.2.0  
**Ratification Date:** 2026-03-12  
**Last Amended:** 2026-03-12

---

## Preamble

This constitution establishes the non-negotiable architectural principles, constraints, and
governance rules for **RiskWeb** — an intranet application for maintaining Fund Account
records stored in Microsoft SQL Server, with Kerberos/SPNEGO Windows SSO authentication
and a mandatory maker-checker (4-eyes) approval workflow.

Every `/speckit.*` command MUST treat this document as the authoritative gate before
proceeding. Deviations require explicit documentation and written justification.

---

## Article I — Technology Stack (NON-NEGOTIABLE)

### I.1 Backend
- **Runtime:** Java 25
- **Framework:** Spring Boot 4
- **Build tool:** Maven
- **Database driver:** Microsoft JDBC Driver for SQL Server (`mssql-jdbc`)
- **ORM:** Spring Data JPA with Hibernate
- **Authentication:** Spring Security with Kerberos/SPNEGO via `spring-security-kerberos`
- **API style:** RESTful JSON over HTTPS

### I.2 Frontend
- **Framework:** React 18+ with TypeScript
- **Build tool:** Vite
- **HTTP client:** Axios
- **State management:** React Context API (no Redux unless justified)
- **Routing:** React Router v6+

#### Component Library Split (NON-NEGOTIABLE)
Two libraries are used with a strict division of responsibility:

| Responsibility | Library |
|---|---|
| App shell — AppBar, Drawer, Typography, layout, spacing, theming | **Material UI (MUI) v6+** |
| Data Grid (account list, change request queue) | **Syncfusion `@syncfusion/ej2-react-grids`** |
| Forms & input controls | **Syncfusion `@syncfusion/ej2-react-inputs`, `@syncfusion/ej2-react-dropdowns`** |
| Charts & dashboards | **Syncfusion `@syncfusion/ej2-react-charts`** |

- Syncfusion is licensed. The license key MUST be registered once at application entry
  point via `registerLicense(key)` before any Syncfusion component is mounted. The key
  MUST be sourced from an environment variable (`VITE_SYNCFUSION_LICENSE_KEY`) and MUST
  NOT be hardcoded or committed to source control.
- MUI and Syncfusion MUST NOT be mixed within the same logical UI unit.
- The MUI theme MUST be configured to align with the AllianceBernstein brand palette
  defined in Article X.

### I.3 Database
- **Engine:** Microsoft SQL Server (existing schema — no DDL migrations)
- **Schema management:** The database schema is pre-existing and owned by the DBA team.
  The application MUST NOT execute DDL statements (CREATE, ALTER, DROP) at runtime.
  Flyway and Liquibase are explicitly PROHIBITED.

### I.4 Deployment
- **Target:** Single deployable fat JAR (`spring-boot-maven-plugin` repackage) serving
  the React static build from `/src/main/resources/static`.
- **No containers.** Docker, Kubernetes, and any container runtime are out of scope.
- **The React build MUST be embedded** in the JAR at build time via the Maven frontend plugin.

---

## Article II — Authentication & Identity (NON-NEGOTIABLE)

- Authentication MUST use **Kerberos/SPNEGO SSO**. There is no login page or form.
  The browser negotiates a Kerberos ticket transparently on the intranet.
- Spring Security MUST be configured with a `SpnegoAuthenticationProcessingFilter` and a
  `KerberosServiceAuthenticationProvider`.
- The authenticated Windows principal (`DOMAIN\username` or `username@DOMAIN`) MUST be
  captured and stored as the canonical user identity for all audit and workflow records.
- Sessions MUST be stateless at the API layer (JWT or Spring Security context per request).
- **No external identity provider, OAuth2, or LDAP form-login** shall be introduced.

---

## Article III — Authorisation & Roles (NON-NEGOTIABLE)

- There is exactly **one role: `ADMIN`**.
- All users with the `ADMIN` role may both **create/edit** (maker) and **approve/reject**
  (checker) account records.
- The ONLY constraint is: **the maker and the checker for any given pending change MUST be
  different Windows login IDs**. The system MUST enforce this at the service layer, not
  only at the UI layer.
- Role assignment is managed externally (Active Directory group membership or equivalent).
  The application reads role claims from the Kerberos principal; it does not manage roles.

---

## Article IV — Maker-Checker Workflow (NON-NEGOTIABLE)

- Every **Create**, **Update**, and **soft-Delete** operation on an Account record MUST go
  through the maker-checker workflow. No direct writes to the live `Account` table are
  permitted from application code except by the approval step.
- Workflow states: `PENDING_APPROVAL` → `APPROVED` | `REJECTED`.
- A separate `AccountChangeRequest` entity MUST track: operation type (`CREATE`, `UPDATE`,
  `DELETE`), the proposed field values (stored as JSON snapshot), maker Windows login,
  maker timestamp, checker Windows login (nullable until actioned), checker timestamp
  (nullable), status, and rejection reason (nullable).
- A maker MAY cancel their own pending request before it is actioned (transition to
  `CANCELLED`).
- A rejected request MAY be re-edited and resubmitted by the original maker, creating a
  new `AccountChangeRequest` record.
- The checker dashboard MUST display all `PENDING_APPROVAL` requests. A checker MUST NOT
  see the approve/reject action buttons on their own requests.
- On `APPROVED`: the live `Account` record is created/updated/soft-deleted atomically
  within the same database transaction as the status update.

---

## Article V — Data Model Constraints (NON-NEGOTIABLE)

### V.1 Account Fields
The `Account` entity MUST map to the existing SQL Server table with exactly these fields:

| Field               | Type         | Notes                              |
|---------------------|--------------|------------------------------------|
| Fund ID             | String       | Primary business key               |
| Fund Name           | String       |                                    |
| ePace ID            | String       | Free text                          |
| Short Name          | String       |                                    |
| Fund Type           | String       |                                    |
| Reporting Frequency | String       |                                    |
| ESMA / 40ACT        | String       |                                    |
| RM Acronym          | String       | Free text                          |
| Active Status       | Boolean/Enum | `ACTIVE` / `INACTIVE`              |
| Created By          | String       | Windows login of original creator  |
| Created On          | Timestamp    | Set at approval of CREATE request  |

### V.2 Soft Delete
- Accounts MUST NOT be hard-deleted. A delete operation sets `Active Status` to `INACTIVE`.
- All list/search queries MUST default to returning only `ACTIVE` records unless the user
  explicitly filters for inactive ones.

### V.3 No Orphan DDL
- The application MUST NOT use `spring.jpa.hibernate.ddl-auto` values of `create`,
  `create-drop`, or `update`. The only permitted value is `validate` or `none`.

---

## Article VI — API Design (NON-NEGOTIABLE)

- All API endpoints MUST be prefixed `/api/v1/`.
- REST conventions:
    - `GET /api/v1/accounts` — paginated list (active by default)
    - `GET /api/v1/accounts/{id}` — single account
    - `POST /api/v1/accounts/requests` — submit a CREATE change request (maker)
    - `PUT /api/v1/accounts/requests/{requestId}` — submit an UPDATE change request (maker)
    - `DELETE /api/v1/accounts/requests/{requestId}` — submit a DELETE change request (maker)
    - `GET /api/v1/accounts/requests` — list pending requests (checker queue)
    - `POST /api/v1/accounts/requests/{requestId}/approve` — approve (checker)
    - `POST /api/v1/accounts/requests/{requestId}/reject` — reject with reason (checker)
    - `DELETE /api/v1/accounts/requests/{requestId}/cancel` — cancel own pending request (maker)
- Every mutating endpoint MUST record the authenticated Windows principal as the actor.
- API responses MUST use a consistent envelope: `{ "data": ..., "error": null }` /
  `{ "data": null, "error": { "code": "...", "message": "..." } }`.
- HTTP 403 MUST be returned (not 404) when a maker attempts to approve their own request.

---

## Article VII — Audit Trail (NON-NEGOTIABLE)

- Every `AccountChangeRequest` row constitutes the immutable audit trail.
- Records in `AccountChangeRequest` MUST NEVER be deleted (no `DELETE` statements from
  application code against this table).
- All timestamps MUST be stored in UTC.
- The `Created By` and `Created On` fields on the live `Account` record reflect the
  original creation maker and the time of approval of the CREATE request — they are
  set once and never overwritten.

---

## Article VIII — Code Quality & Structure (NON-NEGOTIABLE)

### VIII.1 Backend package structure
```
com.{org}.fundaccounts
  ├── config/          # Security, JPA, Web MVC config
  ├── domain/          # JPA entities
  ├── repository/      # Spring Data JPA repositories
  ├── service/         # Business logic (maker-checker enforcement lives here)
  ├── controller/      # REST controllers (thin — delegate to service)
  ├── dto/             # Request/response DTOs (no entity exposure over API)
  └── exception/       # Custom exceptions and global handler (@ControllerAdvice)
```

### VIII.2 Frontend structure
```
src/
  ├── api/             # Axios client + typed API functions
  ├── components/      # Reusable MUI-based components
  ├── pages/           # Route-level page components
  ├── context/         # React Context providers (auth, notifications)
  ├── hooks/           # Custom React hooks
  └── types/           # TypeScript interfaces mirroring backend DTOs
```

### VIII.3 General rules
- Entities MUST NOT be returned directly from controllers — DTOs only.
- Business rule enforcement (maker ≠ checker) MUST live in the service layer and be
  unit-testable in isolation.
- No `System.out.println` or `console.log` in production code; use SLF4J (backend) and
  structured logging.
- All public service methods MUST have Javadoc describing their contract.

---

## Article IX — Testing (NON-NEGOTIABLE)

- **JUnit 5** is the mandatory test framework for all backend tests.
- Unit tests MUST cover:
    - All service-layer methods, especially maker-checker enforcement logic.
    - DTO mapping / validation logic.
- Test class naming convention: `{ClassName}Test` for unit tests.
- Mockito MUST be used to mock repository dependencies in service unit tests.
- Test coverage for the maker-checker enforcement rule (maker ≠ checker) is
  **mandatory** — missing this test is a blocking defect.
- Frontend testing is not mandated in this phase but MUST NOT be broken if added later.

---

## Article X — Brand Guidelines (NON-NEGOTIABLE)

### X.1 Identity
- **Organisation:** AllianceBernstein (AB)
- **Preferred logo:** `logo-1.png` — the standalone AB icon mark (black square with
  white `[A/B]` bracket device). Use this as the primary logo in the app shell header.
- **Compact logo:** `logo-2.png` — the full horizontal lockup (`[A/B]` icon +
  "AllianceBernstein" wordmark on black). Use only when horizontal space permits a
  wider header or on splash/loading screens.
- Logo files MUST be stored in `src/assets/` and referenced via import — never via
  absolute URL strings or public folder paths that could break on base-path changes.
- The logo MUST NOT be recoloured, stretched, or placed on backgrounds that reduce
  contrast below WCAG AA (4.5:1 ratio).

### X.2 Colour Palette
The following palette is authoritative. All MUI theme overrides and Syncfusion theme
variables MUST derive from these tokens.

#### Primary colours
| Token | Name | Hex |
|---|---|---|
| `color-black` | Black | `#000000` |
| `color-white` | White | `#FFFFFF` |
| `color-ab-blue` | AB Blue | `#1E9BD7` |

#### Accent colours (in priority order)
| Token | Name | Hex |
|---|---|---|
| `color-gold` | Gold | `#FFBF27` |
| `color-purple` | Purple | `#5949A7` |
| `color-teal` | Teal | `#1CD8C0` |

#### Usage rules
- **Primary action buttons, links, active states, focus rings:** AB Blue (`#1E9BD7`).
- **Top navigation bar background:** AB Blue (`#1E9BD7`) with White (`#FFFFFF`) text and logo.
- **Left sidebar background:** AB Blue (`#1E9BD7`) with White (`#FFFFFF`) text.
- **Active top nav item highlight:** Gold (`#FFBF27`) pill/badge — White text on Gold background.
- **Main content area background:** Black (`#000000`) with White (`#FFFFFF`) body text.
- **Data emphasis / badges / tags:** follow accent colour order — Gold first, then
  Purple, then Teal. Do not use accent colours for primary interactive elements.
- **Error states:** use MUI default error red — do NOT substitute an AB accent colour.
- **Syncfusion Grid row selection highlight:** AB Blue at 15% opacity (`rgba(30,155,215,0.15)`).

### X.3 MUI Theme Configuration
A single MUI `createTheme` call MUST be defined in `src/theme/abTheme.ts` and provided
via `<ThemeProvider>` at the app root. Minimum overrides required:

```ts
palette: {
  primary:   { main: '#1E9BD7' },
  secondary: { main: '#5949A7' },
  background: { default: '#000000', paper: '#1a1a1a' },
  text: { primary: '#FFFFFF', secondary: '#CCCCCC' },
}
typography: {
  fontFamily: '"Inter", "Helvetica Neue", Arial, sans-serif',
}
```

### X.4 Syncfusion Theme
- Use the Syncfusion **Material** base theme (`@syncfusion/ej2-material-theme`) as the
  starting point.
- Override Syncfusion CSS variables to align header/accent colours with AB Blue.
- Custom Syncfusion overrides MUST live in `src/theme/syncfusionOverrides.css` and be
  imported once in `main.tsx`.

### X.5 Typography
- Primary typeface: **Inter** (Google Fonts or self-hosted). Fallback: Helvetica Neue, Arial.
- Do not introduce additional typefaces without a constitutional amendment.

---

## Article XI — Application Layout & Navigation (NON-NEGOTIABLE)

### XI.1 Application Name
- The application title displayed in the header is **RiskWeb**.
- It MUST appear as plain text in the top navigation bar, immediately to the right of
  the AB logo, in White (`#FFFFFF`), using the Inter typeface at a prominent weight.

### XI.2 Shell Structure
The application shell MUST follow this three-zone layout, matching the reference
screenshot exactly:

```
┌─────────────────────────────────────────────────────────────────┐
│  [Logo] RiskWeb  | Liquidity | SEC | Lux | OpsRisk | RiskIT | Misc  [DOMAIN\user] │
│  ← Top Navigation Bar (AB Blue background, full width)                             │
├──────────────┬──────────────────────────────────────────────────┤
│  Left        │                                                  │
│  Sidebar     │   Main Content Area                             │
│  (AB Blue)   │   (Black background)                            │
│              │                                                  │
│  ◄ collapse  │                                                  │
└──────────────┴──────────────────────────────────────────────────┘
```

- **Top navigation bar:** Fixed, full-width, AB Blue background. Contains logo (left),
  app name (left, after logo), section links (centre/left), and Windows login display (right).
- **Left sidebar:** AB Blue background, collapsible via a `◄` / `►` toggle button.
  Displays the two-level navigation tree for the currently active top nav section.
- **Main content area:** Black (`#000000`) background, fills remaining space.
  Renders the page component for the active sidebar link.

### XI.3 Top Navigation Bar
- The active top nav section MUST be highlighted with a **Gold (`#FFBF27`) filled pill**
  with Black (`#000000`) text — matching the screenshot's active tab style.
- Inactive top nav links MUST be White text on AB Blue background with no decoration.
- Hover state: White text with a subtle underline or lightened background.
- The **Windows login ID** (e.g. `DOMAIN\username`) MUST be displayed as a non-interactive
  chip/badge in the top-right corner. It is display-only — no dropdown, no logout action.
- Clicking a top nav link MUST update the left sidebar to show that section's navigation
  tree, and navigate to that section's default child page.

### XI.4 Left Sidebar Navigation
- The sidebar is **collapsible**. The collapse/expand toggle (`◄` / `►`) MUST be visible
  at the top of the sidebar at all times.
- When collapsed, only the toggle icon is shown — no labels or tree nodes.
- Navigation tree is **two levels deep**: section headers (non-clickable group labels)
  and child page links beneath them.
- The active child link MUST be visually distinguished (e.g. Gold underline or bold
  White text) consistent with the screenshot's "Overview" active state style.
- Sidebar width: `200px` expanded, `48px` collapsed. Transition MUST be animated
  (CSS transition, ~200ms ease).

### XI.5 Navigation Structure
The following is the initial navigation structure. All non-Accounts pages render a
"Work In Progress…" placeholder matching the reference screenshot. This structure MUST
be defined in a single `src/config/navigation.ts` config file — not hardcoded in
component JSX — so sections and links can be added without component changes.

```
Liquidity
  ├── Overview          [placeholder]
  └── Reports           [placeholder]

SEC
  ├── Overview          [placeholder]
  └── Filings           [placeholder]

Lux
  ├── Overview          [placeholder]
  └── Compliance        [placeholder]

OpsRisk
  ├── Overview          [placeholder]
  └── Incidents         [placeholder]

RiskIT
  ├── Overview          [placeholder]
  └── Systems           [placeholder]

Misc
  ├── Overview          [placeholder]
  └── Accounts          [FUNCTIONAL — Fund Account Management]
```

### XI.6 Placeholder Pages
- Every non-functional page MUST render the text **"Work In Progress…"** centred in the
  main content area, in White (`#FFFFFF`) text on the Black background — exactly as
  shown in the reference screenshot.
- Placeholder pages MUST be a single shared component (`<WorkInProgress />`) reused
  across all stub routes — not duplicated per page.

### XI.7 Routing
- React Router v6 nested routes MUST model the two-level navigation:
  `/liquidity/overview`, `/liquidity/reports`, `/sec/overview`, … `/misc/accounts`, etc.
- The default route (`/`) MUST redirect to `/liquidity/overview`.
- The Accounts page (`/misc/accounts`) is the only fully implemented route in this phase.

---

## Governance

- This constitution supersedes all other practices, conventions, and AI defaults.
- Any deviation from Articles I–XI requires:
    1. Explicit written justification committed alongside the code.
    2. A note in the relevant `plan.md` Complexity Tracking section.
- All pull requests / code review MUST verify constitutional compliance before merge.
- Amendments to this constitution require incrementing `Constitution Version` per
  semantic versioning: MAJOR for removals/redefinitions, MINOR for additions,
  PATCH for clarifications.