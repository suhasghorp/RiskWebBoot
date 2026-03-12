# Feature Specification: RiskWeb Core — Fund Account Management

**Feature Branch**: `001-riskweb-core`
**Created**: 2026-03-12
**Status**: Draft
**Input**: Core Fund Account management with maker-checker workflow, Windows SSO authentication, and AllianceBernstein branded shell.

---

## Clarifications

### Session 2026-03-12

- Q: Which fields in the Fund Account creation form are **required** (blocking submission)? → A: Fund ID, Fund Name, Fund Type are required
- Q: When two UPDATE requests for the same account are both approved in sequence, how should conflicting field changes be handled? → A: Allow both approvals; last approval wins; warn checker if multiple pending requests exist for same account
- Q: Which JDBC connection pool should the application use for SQL Server connectivity? → A: HikariCP (Spring Boot default, fastest, recommended)

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Transparent Windows SSO Authentication (Priority: P1)

As an ADMIN user, I am automatically authenticated via Windows SSO when I open RiskWeb in my browser — no login screen.

**Why this priority**: Authentication is the foundation for all other features. Without SSO working, no user can access the application. This is the absolute prerequisite for any functionality.

**Independent Test**: Can be fully tested by opening the application in a corporate intranet browser and verifying that the Windows principal is captured without any login form. Delivers immediate value by proving infrastructure is correctly configured.

**UI Notes** (per Constitution Article X & XI):
- **No login page**: Browser directly loads the main application shell after Kerberos/SPNEGO negotiation.
- **Top navigation bar**: AB Blue background (`#1E9BD7`), displays AB logo (left), "RiskWeb" app name, section links, and **Windows login display** (right).
- **Windows login display**: Shows authenticated principal as `DOMAIN\username` in a non-interactive chip/badge. White text on AB Blue, no hover state, no dropdown, no logout button.

**Acceptance Scenarios**:

1. **Given** I am on a Windows workstation joined to the corporate domain
   **When** I navigate to `https://riskweb.internal` in Chrome/Edge
   **Then** the browser negotiates Kerberos transparently (no prompt) and I land on `/liquidity/overview` showing my Windows login ID in the top-right corner

2. **Given** I am authenticated and viewing any page
   **When** I inspect the top-right corner of the navigation bar
   **Then** I see my Windows principal (e.g., `CORPORATE\jdoe`) displayed as a White-on-AB-Blue chip with no interactive controls

3. **Given** my Kerberos ticket has expired or negotiation fails
   **When** I attempt to access RiskWeb
   **Then** I receive an HTTP 401 Unauthorized response with a clear error message (not a blank page)

4. **Given** I am authenticated
   **When** the backend receives my API request
   **Then** Spring Security context contains my Windows principal and all audit records use this identity

---

### User Story 2 - View Paginated, Searchable Account List (Priority: P1)

As an ADMIN user, I can view a paginated, searchable list of all active Fund Accounts.

**Why this priority**: Viewing accounts is the primary read operation and the entry point for all maker-checker workflows. Without this, users cannot browse, search, or select accounts to edit.

**Independent Test**: Can be fully tested by navigating to `/misc/accounts` and verifying the Syncfusion Grid displays active accounts with search/filter capabilities. Delivers immediate value by providing data visibility.

**UI Notes** (per Constitution Articles X & XI):
- **Route**: `/misc/accounts` (under Misc → Accounts in left sidebar)
- **Main content area**: Black background (`#000000`), White body text (`#FFFFFF`)
- **Data Grid**: Syncfusion `GridComponent` with Material theme overrides aligning to AB Blue accents
- **Grid row selection highlight**: AB Blue at 15% opacity (`rgba(30,155,215,0.15)`)
- **Active top nav**: "Misc" pill highlighted with Gold background (`#FFBF27`), Black text
- **Active left sidebar link**: "Accounts" with Gold underline or bold White text

**Acceptance Scenarios**:

1. **Given** I am authenticated
   **When** I click "Misc" in the top navigation and then "Accounts" in the left sidebar
   **Then** I navigate to `/misc/accounts` and see a Syncfusion Grid displaying all active Fund Accounts with columns: Fund ID, Fund Name, ePace ID, Short Name, Fund Type, Reporting Frequency, ESMA/40ACT, RM Acronym, Active Status

2. **Given** the Accounts page is loaded with 150 active accounts
   **When** I view the grid
   **Then** pagination controls are visible, showing 10-50 rows per page (configurable), and I can navigate between pages

3. **Given** I am viewing the account list
   **When** I type "Equity" into the search box
   **Then** the grid filters in real-time to show only accounts where Fund Name, Fund Type, or Short Name contains "Equity"

4. **Given** I am viewing the account list
   **When** I click a column header (e.g., "Fund Name")
   **Then** the grid sorts by that column in ascending order; clicking again sorts descending

5. **Given** the database contains both active and inactive accounts
   **When** I load the Accounts page without applying filters
   **Then** only accounts with `Active Status = ACTIVE` are displayed by default (soft-deleted accounts are hidden unless I explicitly filter for inactive)

---

### User Story 3 - Submit New Fund Account for Approval (Priority: P1)

As an ADMIN maker, I can submit a new Fund Account for approval.

**Why this priority**: Creating new accounts is a core business capability. This story establishes the entire maker-checker workflow pattern that will be reused for updates and deletions.

**Independent Test**: Can be fully tested by clicking "Create Account", filling the form, submitting, and verifying a pending `AccountChangeRequest` record appears in the checker queue. Delivers immediate value by enabling account creation workflow.

**UI Notes** (per Constitution Articles X & XI):
- **Trigger**: "Create Account" button above the grid (AB Blue `#1E9BD7` primary button, White text)
- **Form modal/dialog**: Syncfusion form inputs (`TextBoxComponent`, `DropDownListComponent`) on Black background
- **Form labels**: White text, required fields marked with asterisk
- **Submit button**: AB Blue primary button labeled "Submit for Approval"
- **Cancel button**: Secondary button (outlined, White border on Black background)

**Acceptance Scenarios**:

1. **Given** I am on the Accounts page
   **When** I click the "Create Account" button
   **Then** a modal dialog opens with a blank form containing fields: Fund ID, Fund Name, ePace ID, Short Name, Fund Type, Reporting Frequency, ESMA/40ACT, RM Acronym

2. **Given** the create form is open
   **When** I fill the required fields (Fund ID, Fund Name, Fund Type) and click "Submit for Approval"
   **Then** a POST request is sent to `/api/v1/accounts/requests` with operation type `CREATE`, the form data as a JSON snapshot, my Windows login as maker, status `PENDING_APPROVAL`, and a success message confirms submission

3. **Given** I submitted a create request
   **When** I return to the account list
   **Then** the new account does NOT appear in the live grid (it is pending approval), but I see a notification: "Account creation request submitted for approval"

4. **Given** the create form is open
   **When** I leave any of the required fields (Fund ID, Fund Name, Fund Type) blank and click submit
   **Then** inline validation errors appear in White text below each empty required field (e.g., "Fund ID is required"), and the submit button remains enabled for correction

5. **Given** I am filling the create form
   **When** I click "Cancel"
   **Then** the modal closes, discarding unsaved data, and I return to the account list unchanged

---

### User Story 4 - Submit Edit to Existing Account for Approval (Priority: P2)

As an ADMIN maker, I can submit an edit to an existing Fund Account for approval.

**Why this priority**: Editing accounts is essential for maintaining data accuracy. While lower priority than creation (because you need accounts to exist first), it's still a core workflow.

**Independent Test**: Can be fully tested by selecting an account, clicking "Edit", modifying fields, and submitting. Verify the live record remains unchanged until approval. Delivers value by enabling data maintenance.

**UI Notes** (per Constitution Articles X & XI):
- **Trigger**: "Edit" action button in each grid row (icon button or context menu)
- **Form modal**: Pre-populated with current account values
- **Changed fields highlight**: Optional visual indicator (e.g., light Gold `#FFBF27` border) on modified inputs
- **Submit button**: "Submit for Approval" (AB Blue primary)

**Acceptance Scenarios**:

1. **Given** I am viewing the account list with at least one active account
   **When** I click the "Edit" button for Fund ID "ABC123"
   **Then** a modal opens with the form pre-populated with all current field values for ABC123

2. **Given** the edit form is open for ABC123
   **When** I change "Fund Name" from "Old Name" to "New Name" and click "Submit for Approval"
   **Then** a PUT request is sent to `/api/v1/accounts/requests/{requestId}` with operation type `UPDATE`, the modified field(s) in the JSON snapshot, my Windows login as maker, and the request transitions to `PENDING_APPROVAL`

3. **Given** I submitted an edit request for ABC123
   **When** I return to the account list and view ABC123
   **Then** the grid still shows "Old Name" (the live record is unchanged until approved)

4. **Given** the edit form is open
   **When** I modify a field and then click "Cancel"
   **Then** the modal closes without creating a change request, and the account remains unchanged

5. **Given** I am editing ABC123 and change only the "Reporting Frequency" field
   **When** I submit
   **Then** the JSON snapshot in `AccountChangeRequest` includes only the changed field(s) plus the full context required for the checker to review

---

### User Story 5 - Submit Deactivation (Soft-Delete) for Approval (Priority: P2)

As an ADMIN maker, I can submit a deactivation (soft-delete) of a Fund Account for approval.

**Why this priority**: Deactivation is less frequent than creation or editing, but still necessary for data lifecycle management. Placed at P2 because it follows the same workflow pattern as create/edit.

**Independent Test**: Can be fully tested by clicking "Deactivate" on an active account, confirming, and verifying the account remains active until the request is approved. Delivers value by enabling safe account archival.

**UI Notes** (per Constitution Articles X & XI):
- **Trigger**: "Deactivate" action button in each grid row (icon button with warning color — use MUI default error red, not AB accent colors per Article X.2)
- **Confirmation dialog**: Syncfusion dialog or MUI `Dialog` with warning text: "Deactivating this account will set its status to INACTIVE. This action requires approval."
- **Confirm button**: Error red background, White text, labeled "Submit for Approval"
- **Cancel button**: Secondary outlined button

**Acceptance Scenarios**:

1. **Given** I am viewing the account list
   **When** I click the "Deactivate" button for Fund ID "XYZ789"
   **Then** a confirmation dialog opens with the message: "Deactivating XYZ789 will set its status to INACTIVE. This action requires approval. Continue?"

2. **Given** the deactivation confirmation dialog is open
   **When** I click "Submit for Approval"
   **Then** a DELETE request is sent to `/api/v1/accounts/requests/{requestId}` with operation type `DELETE`, the JSON snapshot captures the current account state, my Windows login as maker, status `PENDING_APPROVAL`, and the dialog closes with a success message

3. **Given** I submitted a deactivation request for XYZ789
   **When** I view the account list
   **Then** XYZ789 is still displayed as `ACTIVE` (the soft-delete has not yet occurred)

4. **Given** the deactivation confirmation dialog is open
   **When** I click "Cancel"
   **Then** the dialog closes and no change request is created

5. **Given** a deactivation request for XYZ789 is approved
   **When** I reload the account list
   **Then** XYZ789 no longer appears in the default view (which shows only `ACTIVE` accounts), but if I apply a filter to show inactive accounts, XYZ789 appears with `Active Status = INACTIVE`

---

### User Story 6 - Cancel Own Pending Change Request (Priority: P3)

As an ADMIN maker, I can cancel my own pending change request before it is approved.

**Why this priority**: Allows makers to retract mistakes without waiting for checker action. Lower priority because it's a convenience feature — the workflow functions without it, but it improves user experience.

**Independent Test**: Can be fully tested by submitting a change request, navigating to "My Pending Requests", and clicking "Cancel". Verify the request transitions to `CANCELLED` and disappears from both maker and checker queues. Delivers value by reducing unnecessary checker workload.

**UI Notes** (per Constitution Articles X & XI):
- **Location**: "My Pending Requests" link in left sidebar under Misc → Accounts (optional sub-page) OR a filtered view toggle on the Accounts page
- **Trigger**: "Cancel" button next to each of my own pending requests
- **Cancel button**: Secondary button (outlined White border), labeled "Cancel Request"
- **Confirmation dialog**: Optional, but recommended: "Are you sure you want to cancel this request?"

**Acceptance Scenarios**:

1. **Given** I submitted a CREATE request for a new account 5 minutes ago
   **When** I navigate to "My Pending Requests" (or filter the requests queue to show only my requests)
   **Then** I see my pending CREATE request with status `PENDING_APPROVAL` and a "Cancel Request" button

2. **Given** I am viewing my own pending CREATE request
   **When** I click "Cancel Request" and confirm
   **Then** a DELETE request is sent to `/api/v1/accounts/requests/{requestId}/cancel`, the status transitions to `CANCELLED`, and the request disappears from the pending queue

3. **Given** I cancelled my own CREATE request
   **When** a checker views the pending requests queue
   **Then** the cancelled request does NOT appear (it is no longer actionable)

4. **Given** another user (e.g., `CORPORATE\alice`) submitted a pending request
   **When** I (e.g., `CORPORATE\bob`) view the pending requests queue
   **Then** I do NOT see a "Cancel" button on Alice's request (only the maker can cancel their own request)

5. **Given** I submitted a request and it was already approved by a checker
   **When** I view the request history
   **Then** the "Cancel" button is no longer available (requests can only be cancelled while `PENDING_APPROVAL`)

---

### User Story 7 - View All Pending Change Requests (Checker Queue) (Priority: P1)

As an ADMIN checker, I can view all pending change requests submitted by other users.

**Why this priority**: The checker queue is the heart of the 4-eyes approval workflow. Without this, the entire maker-checker pattern cannot function. This is foundational for approval operations.

**Independent Test**: Can be fully tested by navigating to the "Pending Approvals" page and verifying all `PENDING_APPROVAL` requests from other users are displayed. Delivers immediate value by enabling oversight and audit.

**UI Notes** (per Constitution Articles X & XI):
- **Route**: `/misc/accounts/pending` OR a tab/view toggle on the Accounts page
- **Left sidebar**: Add "Pending Approvals" link under Misc → Accounts
- **Data Grid**: Syncfusion Grid with columns: Request ID, Operation (CREATE/UPDATE/DELETE), Fund ID, Fund Name (or "New Account"), Maker, Submitted On, Actions (Approve/Reject buttons)
- **Approve button**: AB Blue primary button (disabled for own requests)
- **Reject button**: Error red outlined button (disabled for own requests)
- **Own requests**: Rows where `maker = my Windows login` are displayed but approve/reject buttons are disabled with a tooltip: "You cannot approve your own request"

**Acceptance Scenarios**:

1. **Given** I am authenticated as `CORPORATE\checker1`
   **When** I navigate to "Pending Approvals" under Misc → Accounts
   **Then** I see a Syncfusion Grid listing all change requests with status `PENDING_APPROVAL`, including requests from `CORPORATE\maker1`, `CORPORATE\maker2`, etc., but NOT requests I submitted myself (or if they appear, the action buttons are disabled)

2. **Given** the pending approvals grid is displayed with 3 requests: 1 CREATE, 1 UPDATE, 1 DELETE
   **When** I view the grid
   **Then** each row shows: Request ID, operation type badge (color-coded: Gold for CREATE, Purple for UPDATE, Teal for DELETE per Article X.2 accent priority), Fund ID, Fund Name (or "New Account" for CREATE), maker Windows login, submitted timestamp

3. **Given** I am viewing a pending UPDATE request for Fund ID "ABC123"
   **When** I click the row to expand details (or a "View Details" button)
   **Then** I see the current live values vs. the proposed changes in a side-by-side comparison or highlighted diff format

4. **Given** there is a pending CREATE request submitted by `CORPORATE\maker1` (and I am `CORPORATE\checker1`)
   **When** I view the request row
   **Then** the "Approve" and "Reject" buttons are enabled (not disabled)

5. **Given** there is a pending UPDATE request submitted by me (`CORPORATE\checker1`)
   **When** I view the request row in the pending queue
   **Then** the "Approve" and "Reject" buttons are disabled, and hovering shows a tooltip: "You cannot approve your own request"

---

### User Story 8 - Approve Pending Change Request (Priority: P1)

As an ADMIN checker, I can approve a pending change request, causing the live Account record to be updated.

**Why this priority**: Approval is the final step that makes the maker-checker workflow functional. Without this, all submitted requests remain in limbo. This is critical path.

**Independent Test**: Can be fully tested by approving a CREATE request and verifying the new account appears in the live account list. Delivers immediate value by completing the workflow loop.

**UI Notes** (per Constitution Articles X & XI):
- **Trigger**: "Approve" button in the pending requests grid row
- **Confirmation dialog**: Optional but recommended: "Approve this CREATE/UPDATE/DELETE request? This will update the live Account record."
- **Approve button**: AB Blue primary button, labeled "Approve"
- **Success feedback**: Success message (MUI Snackbar or Syncfusion Toast): "Request approved. Live account updated."

**Acceptance Scenarios**:

1. **Given** I am viewing a pending CREATE request for a new account "DEF456"
   **When** I click "Approve" and confirm
   **Then** a POST request is sent to `/api/v1/accounts/requests/{requestId}/approve`, the request status transitions to `APPROVED`, the live `Account` table inserts a new row with `Created By = maker Windows login` and `Created On = approval timestamp`, and I see a success message

2. **Given** I approved the CREATE request for DEF456
   **When** I navigate back to the main Accounts page
   **Then** DEF456 now appears in the active accounts grid

3. **Given** I am viewing a pending UPDATE request for Fund ID "ABC123" that changes "Fund Name" from "Old Name" to "New Name"
   **When** I approve the request
   **Then** the live `Account` record for ABC123 is updated with "Fund Name = New Name" within the same database transaction as the status change, and `Created By` / `Created On` fields remain unchanged (they reflect the original creation, not the update)

4. **Given** I am viewing a pending DELETE request for Fund ID "XYZ789"
   **When** I approve the request
   **Then** the live `Account` record for XYZ789 has `Active Status` set to `INACTIVE` (soft-delete), and XYZ789 no longer appears in the default account list (which filters for `ACTIVE` only)

5. **Given** the approval operation fails (e.g., database constraint violation)
   **When** the backend attempts to update the live record
   **Then** the transaction is rolled back, the request status remains `PENDING_APPROVAL`, and I receive an error message with the reason (logged with SLF4J, not `System.out.println`)

6. **Given** I approved a request
   **When** I view the `AccountChangeRequest` table
   **Then** the record shows: `status = APPROVED`, `checker = my Windows login`, `checker_timestamp = approval time`, and the record is never deleted (immutable audit trail per Article VII)

---

### User Story 9 - Reject Pending Change Request with Reason (Priority: P2)

As an ADMIN checker, I can reject a pending change request with a reason, which allows the maker to resubmit.

**Why this priority**: Rejection with feedback is important for data quality and user collaboration, but the application can function without it initially (makers can just cancel and resubmit). Placed at P2 for this reason.

**Independent Test**: Can be fully tested by clicking "Reject", entering a reason, and verifying the request transitions to `REJECTED` with the reason stored. Maker can then view the reason and resubmit. Delivers value by enabling feedback loop.

**UI Notes** (per Constitution Articles X & XI):
- **Trigger**: "Reject" button in the pending requests grid row (error red outlined button)
- **Rejection dialog**: Modal with text area for rejection reason (required field), White text on Black background
- **Text area label**: "Reason for Rejection (required)"
- **Submit button**: Error red primary button, labeled "Reject Request"
- **Cancel button**: Secondary outlined button

**Acceptance Scenarios**:

1. **Given** I am viewing a pending CREATE request
   **When** I click "Reject"
   **Then** a modal opens with a required text area labeled "Reason for Rejection"

2. **Given** the rejection dialog is open
   **When** I enter "Fund ID already exists in external system" and click "Reject Request"
   **Then** a POST request is sent to `/api/v1/accounts/requests/{requestId}/reject` with the rejection reason in the body, the status transitions to `REJECTED`, `checker = my Windows login`, `rejection_reason = the entered text`, and I see a success message

3. **Given** I rejected a CREATE request
   **When** the maker views their request history
   **Then** the request shows status `REJECTED` with my rejection reason displayed in White text

4. **Given** a maker's request was rejected with reason "Missing required ePace ID"
   **When** the maker clicks "Resubmit" (or creates a new request)
   **Then** a new `AccountChangeRequest` record is created (the old rejected request remains in the audit trail per Article VII)

5. **Given** the rejection dialog is open
   **When** I leave the reason field blank and click "Reject Request"
   **Then** inline validation error appears: "Rejection reason is required" (White text), and the submit button remains enabled for correction

6. **Given** the rejection dialog is open
   **When** I click "Cancel"
   **Then** the dialog closes, the request remains `PENDING_APPROVAL`, and no change is made

---

### User Story 10 - Navigation Shell with Placeholder Pages (Priority: P1)

As any ADMIN user, I can see all sections of RiskWeb in the top navigation; non-Accounts sections show a "Work In Progress" placeholder.

**Why this priority**: The application shell (navigation, branding, layout) is the structural foundation for the entire application. Without it, users cannot navigate or experience the branded interface. This must be built first.

**Independent Test**: Can be fully tested by loading the application and clicking each top nav section (Liquidity, SEC, Lux, OpsRisk, RiskIT, Misc). Verify the shell structure matches the constitution screenshot exactly and placeholder pages render correctly. Delivers immediate value by establishing the application UX framework.

**UI Notes** (per Constitution Articles X & XI — this story implements Article XI in full):
- **Top navigation bar**: Fixed, full-width, AB Blue background (`#1E9BD7`), contains:
  - AB logo (`logo-1.png`, left)
  - "RiskWeb" app name (White text, Inter font, left after logo)
  - Section links: Liquidity | SEC | Lux | OpsRisk | RiskIT | Misc (White text, centered/left)
  - Windows login chip (White text, right)
- **Active top nav highlight**: Gold pill (`#FFBF27`) with Black text (e.g., active section "Misc")
- **Left sidebar**: AB Blue background (`#1E9BD7`), collapsible via `◄` / `►` toggle, width `200px` expanded / `48px` collapsed, animated transition ~200ms ease
- **Sidebar navigation tree**: Two levels — section headers (non-clickable) + child page links (clickable)
- **Active child link**: Gold underline or bold White text
- **Main content area**: Black background (`#000000`), White body text (`#FFFFFF`)
- **Placeholder pages**: All non-Accounts routes render `<WorkInProgress />` component with centered text "Work In Progress…" (White on Black)

**Acceptance Scenarios**:

1. **Given** I am authenticated and land on the default route `/`
   **When** the application loads
   **Then** I am redirected to `/liquidity/overview`, the "Liquidity" top nav pill is highlighted with Gold background, the left sidebar shows the Liquidity navigation tree (Overview, Reports), and the main content area displays "Work In Progress…" centered in White text

2. **Given** I am on any page
   **When** I view the top navigation bar
   **Then** I see (left to right): AB logo (`logo-1.png`), "RiskWeb" in White text, section links (Liquidity | SEC | Lux | OpsRisk | RiskIT | Misc) in White, and my Windows login (e.g., `CORPORATE\jdoe`) in a White chip on the right

3. **Given** I am viewing `/liquidity/overview`
   **When** I click "SEC" in the top navigation
   **Then** I navigate to `/sec/overview`, the "SEC" pill is highlighted with Gold background (`#FFBF27`) and Black text, the left sidebar updates to show the SEC navigation tree (Overview, Filings), and the main content area shows "Work In Progress…"

4. **Given** I am on any page
   **When** I click the `◄` collapse toggle in the left sidebar
   **Then** the sidebar animates to `48px` width over ~200ms, only the toggle icon `►` is visible, and the main content area expands to fill the space

5. **Given** the left sidebar is collapsed
   **When** I click the `►` expand toggle
   **Then** the sidebar animates to `200px` width, the navigation tree labels become visible, and the main content area contracts

6. **Given** I am on the "Misc" section
   **When** I click "Accounts" in the left sidebar
   **Then** I navigate to `/misc/accounts`, the "Accounts" link is highlighted with Gold underline or bold White text, and the main content area displays the Syncfusion Grid with live accounts (NOT a placeholder)

7. **Given** I am viewing any page
   **When** I inspect the navigation structure
   **Then** the navigation config matches the structure defined in Article XI.5:
   - Liquidity → Overview (placeholder), Reports (placeholder)
   - SEC → Overview (placeholder), Filings (placeholder)
   - Lux → Overview (placeholder), Compliance (placeholder)
   - OpsRisk → Overview (placeholder), Incidents (placeholder)
   - RiskIT → Overview (placeholder), Systems (placeholder)
   - Misc → Overview (placeholder), Accounts (FUNCTIONAL)

8. **Given** I navigate to any placeholder route (e.g., `/liquidity/reports`, `/sec/filings`, `/lux/compliance`)
   **When** the page loads
   **Then** the main content area renders the shared `<WorkInProgress />` component with the text "Work In Progress…" centered vertically and horizontally in White (`#FFFFFF`) text on the Black (`#000000`) background — exactly as shown in the constitution's reference screenshot

9. **Given** I am viewing the application shell
   **When** I inspect the color palette usage
   **Then** I verify:
   - Top nav background: AB Blue `#1E9BD7`
   - Left sidebar background: AB Blue `#1E9BD7`
   - Main content area background: Black `#000000`
   - Active top nav pill: Gold `#FFBF27` background, Black text
   - Inactive top nav links: White text, no decoration
   - Windows login chip: White text on AB Blue
   - All body text: White `#FFFFFF`

10. **Given** I am viewing the application
    **When** I inspect the logo
    **Then** the AB logo (`logo-1.png` — the standalone AB icon mark) is displayed in the top-left corner, imported from `src/assets/`, not recoloured or stretched, and placed on the AB Blue background with sufficient contrast (WCAG AA 4.5:1 ratio)

---

### Edge Cases

- **What happens when Kerberos negotiation fails?**
  → The application returns HTTP 401 with a user-friendly error message. No fallback login form is displayed (per Article II).

- **What happens when a maker and checker are the same Windows login?**
  → The service layer enforces maker ≠ checker at the business logic level. If the maker attempts to approve their own request, the API returns HTTP 403 with error: `{ "code": "FORBIDDEN", "message": "You cannot approve your own request" }`. The UI disables the approve/reject buttons for own requests (defensive UX).

- **What happens when a checker tries to approve a request that was already approved by another checker?**
  → The backend detects the status is no longer `PENDING_APPROVAL` and returns HTTP 409 Conflict with an error message. The UI should refresh the pending queue to reflect the current state.

- **What happens when an account deactivation is approved while the account is referenced in a pending CREATE/UPDATE request?**
  → This is a business logic decision: either (a) reject the deactivation if there are pending requests referencing the account, or (b) allow the deactivation and automatically reject dependent requests. Document the chosen approach in the implementation plan.

- **What happens when the same account is edited by two makers simultaneously, creating two pending UPDATE requests?**
  → Both requests are created independently. The checker can approve them in sequence. If both requests modify the same field to different values, the later approval wins (last-write-wins). The checker UI MUST display a warning badge when viewing a request if other pending UPDATE requests exist for the same account, showing: "Warning: 1 other pending request exists for this account. Review carefully to avoid conflicts."

- **What happens when a user's Kerberos ticket expires mid-session?**
  → The next API call will fail with HTTP 401. The frontend should detect this and display a message: "Your session has expired. Please refresh the page to re-authenticate."

- **What happens when the Syncfusion license key is missing or invalid?**
  → Syncfusion components will display a license warning banner. The application should fail fast at startup if `VITE_SYNCFUSION_LICENSE_KEY` is not set, logging an error and preventing component mount.

- **What happens when the database is unavailable during an approval operation?**
  → The transaction fails, the backend logs the error via SLF4J, and the API returns HTTP 500 with a generic error message. The request status remains `PENDING_APPROVAL` (no partial writes due to transaction rollback per Article IV).

- **What happens when an inactive account is displayed (after applying the "show inactive" filter)?**
  → The row is displayed in the grid with `Active Status = INACTIVE`. Optionally, the row can be styled differently (e.g., dimmed text or strikethrough Fund Name) to visually distinguish inactive accounts.

- **What happens when a user directly navigates to `/api/v1/accounts` without authentication?**
  → Spring Security intercepts the request before it reaches the controller, returning HTTP 401 Unauthorized (no data leakage).

---

## Requirements *(mandatory)*

### Functional Requirements

#### Authentication & Identity (Article II)
- **FR-001**: System MUST authenticate users via Kerberos/SPNEGO SSO without a login page or form
- **FR-002**: System MUST capture the Windows principal (`DOMAIN\username`) and store it as the canonical user identity
- **FR-003**: System MUST display the authenticated Windows login ID in the top-right corner of the navigation bar as a non-interactive chip
- **FR-004**: System MUST return HTTP 401 Unauthorized if Kerberos negotiation fails, with a clear error message (no blank page)

#### Authorisation (Article III)
- **FR-005**: System MUST assign the `ADMIN` role to all authenticated users (role assignment managed externally via AD group membership)
- **FR-006**: System MUST enforce the maker-checker constraint at the service layer: the maker and checker for any pending change MUST be different Windows login IDs
- **FR-007**: System MUST return HTTP 403 Forbidden if a user attempts to approve/reject their own request

#### Maker-Checker Workflow (Article IV)
- **FR-008**: System MUST route all CREATE, UPDATE, and DELETE operations on the `Account` entity through the maker-checker workflow (no direct writes to the `Account` table except via approval)
- **FR-009**: System MUST create an `AccountChangeRequest` record for every maker action, storing: operation type (`CREATE`, `UPDATE`, `DELETE`), JSON snapshot of proposed field values, maker Windows login, maker timestamp, status (`PENDING_APPROVAL`)
- **FR-010**: System MUST allow a maker to cancel their own pending request (transition to `CANCELLED`) before it is actioned
- **FR-011**: System MUST allow a checker to approve a pending request, atomically updating the live `Account` record and the `AccountChangeRequest` status to `APPROVED` within the same database transaction
- **FR-012**: System MUST allow a checker to reject a pending request with a mandatory rejection reason, transitioning status to `REJECTED`
- **FR-013**: System MUST set `Created By` and `Created On` fields on the `Account` record only on CREATE approval (never overwritten on UPDATE)
- **FR-014**: System MUST hide approve/reject action buttons on the checker's own pending requests in the UI (defensive UX, enforced at backend via FR-006)
- **FR-014a**: System MUST display a warning badge in the checker UI when viewing an UPDATE request if other `PENDING_APPROVAL` UPDATE requests exist for the same Account ID, with message: "Warning: N other pending request(s) exist for this account. Review carefully to avoid conflicts."
- **FR-014b**: System MUST apply last-write-wins conflict resolution when multiple UPDATE requests for the same account are approved sequentially (later approval overwrites earlier changes to the same field)

#### Account Data Model (Article V)
- **FR-015**: System MUST map the `Account` entity to the existing SQL Server table with exactly these fields: Fund ID (String), Fund Name (String), ePace ID (String), Short Name (String), Fund Type (String), Reporting Frequency (String), ESMA/40ACT (String), RM Acronym (String), Active Status (Boolean/Enum), Created By (String), Created On (Timestamp)
- **FR-015a**: System MUST enforce that Fund ID, Fund Name, and Fund Type are required fields (NOT NULL) for CREATE operations; all other account fields are optional
- **FR-015b**: System MUST use HikariCP as the JDBC connection pool for SQL Server (Spring Boot default, transitive dependency via `spring-boot-starter-data-jpa`)
- **FR-016**: System MUST implement soft delete: DELETE operations set `Active Status = INACTIVE` (no hard deletes)
- **FR-017**: System MUST default all list/search queries to return only `ACTIVE` accounts unless the user explicitly filters for inactive
- **FR-018**: System MUST use `spring.jpa.hibernate.ddl-auto = validate` or `none` (no `create`, `create-drop`, `update` per Article V.3)

#### API Design (Article VI)
- **FR-019**: System MUST prefix all API endpoints with `/api/v1/`
- **FR-020**: System MUST implement the following REST endpoints:
  - `GET /api/v1/accounts` — paginated list of accounts (active by default)
  - `GET /api/v1/accounts/{id}` — single account detail
  - `POST /api/v1/accounts/requests` — submit CREATE change request
  - `PUT /api/v1/accounts/requests/{requestId}` — submit UPDATE change request
  - `DELETE /api/v1/accounts/requests/{requestId}` — submit DELETE change request
  - `GET /api/v1/accounts/requests` — list pending requests (checker queue)
  - `POST /api/v1/accounts/requests/{requestId}/approve` — approve request
  - `POST /api/v1/accounts/requests/{requestId}/reject` — reject with reason
  - `DELETE /api/v1/accounts/requests/{requestId}/cancel` — cancel own request
- **FR-021**: System MUST use consistent API response envelope: `{ "data": ..., "error": null }` for success, `{ "data": null, "error": { "code": "...", "message": "..." } }` for errors
- **FR-022**: System MUST return HTTP 403 (not 404) when a maker attempts to approve their own request

#### Audit Trail (Article VII)
- **FR-023**: System MUST preserve all `AccountChangeRequest` records as the immutable audit trail (no DELETE statements against this table)
- **FR-024**: System MUST store all timestamps in UTC
- **FR-025**: System MUST capture maker Windows login, maker timestamp, checker Windows login (nullable), checker timestamp (nullable), status, and rejection reason (nullable) on every `AccountChangeRequest` record

#### UI Shell & Navigation (Article XI)
- **FR-026**: System MUST implement a three-zone layout: top navigation bar (AB Blue), left sidebar (AB Blue, collapsible), main content area (Black)
- **FR-027**: System MUST display the AB logo (`logo-1.png`), "RiskWeb" app name, section links, and Windows login chip in the top navigation bar
- **FR-028**: System MUST highlight the active top nav section with a Gold pill (`#FFBF27`) background and Black text
- **FR-029**: System MUST render the left sidebar with a two-level navigation tree and a collapse/expand toggle (`◄` / `►`)
- **FR-030**: System MUST collapse the sidebar to `48px` width (toggle only) and expand to `200px` width (full tree) with a ~200ms CSS transition
- **FR-031**: System MUST render all non-Accounts routes with a shared `<WorkInProgress />` component displaying "Work In Progress…" (White text on Black background)
- **FR-032**: System MUST define the navigation structure in a single `src/config/navigation.ts` config file (not hardcoded in JSX)
- **FR-033**: System MUST redirect the default route `/` to `/liquidity/overview`

#### Branding (Article X)
- **FR-034**: System MUST use the AllianceBernstein color palette: AB Blue `#1E9BD7` (primary), Gold `#FFBF27` (accent 1), Purple `#5949A7` (accent 2), Teal `#1CD8C0` (accent 3), Black `#000000`, White `#FFFFFF`
- **FR-035**: System MUST configure the MUI theme with primary color AB Blue, background default Black, text primary White (per Article X.3)
- **FR-036**: System MUST override Syncfusion CSS variables to align with AB Blue accents (per Article X.4)
- **FR-037**: System MUST use the Inter typeface (Google Fonts or self-hosted, fallback Helvetica Neue, Arial)
- **FR-038**: System MUST store logo files (`logo-1.png`, `logo-2.png`) in `src/assets/` and import via ES modules (not public folder paths)
- **FR-039**: System MUST use MUI default error red for error states (not AB accent colors)
- **FR-040**: System MUST use AB Blue at 15% opacity (`rgba(30,155,215,0.15)`) for Syncfusion Grid row selection highlight

#### Testing (Article IX)
- **FR-041**: System MUST use JUnit 5 as the test framework for all backend tests
- **FR-042**: System MUST include unit tests for all service-layer methods, especially maker-checker enforcement logic (maker ≠ checker)
- **FR-043**: System MUST use Mockito to mock repository dependencies in service unit tests
- **FR-044**: System MUST include a test verifying that a maker cannot approve their own request (test MUST fail if this logic is removed)

### Key Entities *(data model)*

- **Account**: Live Fund Account record — Fund ID (String, **required**), Fund Name (String, **required**), Fund Type (String, **required**), ePace ID (String, optional), Short Name (String, optional), Reporting Frequency (String, optional), ESMA/40ACT (String, optional), RM Acronym (String, optional), Active Status (Boolean/Enum, set by system), Created By (String, set by system on approval), Created On (Timestamp, set by system on approval). Mapped to existing SQL Server table (no DDL).

- **AccountChangeRequest**: Maker-checker workflow record — Request ID (PK), Operation Type (`CREATE`, `UPDATE`, `DELETE`), Account ID (nullable for CREATE), JSON Snapshot (proposed field values), Maker Windows Login, Maker Timestamp, Checker Windows Login (nullable), Checker Timestamp (nullable), Status (`PENDING_APPROVAL`, `APPROVED`, `REJECTED`, `CANCELLED`), Rejection Reason (nullable). Immutable audit trail — never deleted.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Authenticated users can land on the RiskWeb application without seeing a login form, and their Windows login ID is displayed in the top-right corner (SSO transparency)

- **SC-002**: Users can view a paginated, searchable list of active Fund Accounts with real-time filtering and sorting in under 2 seconds for datasets of up to 500 accounts

- **SC-003**: Makers can submit CREATE, UPDATE, and DELETE requests, and these requests appear in the checker queue without modifying the live Account table until approved (workflow isolation)

- **SC-004**: Checkers can approve or reject requests submitted by other users, and the system enforces the maker ≠ checker rule at the service layer (no bypass via API)

- **SC-005**: 100% of maker-checker enforcement logic is covered by unit tests, including the test verifying that a maker cannot approve their own request

- **SC-006**: All navigation sections (Liquidity, SEC, Lux, OpsRisk, RiskIT, Misc) are accessible via the top navigation bar, and non-Accounts pages display the "Work In Progress…" placeholder matching the constitution's reference screenshot

- **SC-007**: The application shell (top nav, sidebar, main content area) matches the AllianceBernstein brand guidelines with exact color values: AB Blue `#1E9BD7`, Gold `#FFBF27`, Black `#000000`, White `#FFFFFF`

- **SC-008**: All `AccountChangeRequest` records are preserved as an immutable audit trail (zero deletions from this table during normal operation)

- **SC-009**: The Syncfusion license key is registered at application startup via an environment variable, and Syncfusion components render without license warnings

- **SC-010**: The application runs as a single deployable fat JAR with the React build embedded in `/src/main/resources/static`, and navigating to `https://riskweb.internal/` serves the React app from the Spring Boot server
