# Tasks: RiskWeb Core — Fund Account Management

**Input**: Design documents from `/specs/001-riskweb-core/`
**Prerequisites**: plan.md (required), spec.md (required), research.md, data-model.md, contracts/api-v1.md

**Tests**: Backend unit tests are mandatory per Constitution Article IX. Frontend testing not mandated in this phase.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

---

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and basic structure

- [ ] T001 Create Maven project structure with parent pom.xml and Spring Boot 4 starter dependencies
- [ ] T002 Create frontend directory structure with package.json, vite.config.ts, tsconfig.json
- [ ] T003 [P] Configure frontend-maven-plugin in pom.xml to build React app and copy to src/main/resources/static/
- [ ] T004 [P] Add application.yml with datasource, JPA, and Kerberos security configuration templates
- [ ] T005 [P] Add Syncfusion and MUI dependencies to frontend/package.json
- [ ] T006 [P] Create .gitignore files for Java and React projects
- [ ] T007 [P] Add environment variable templates for DB_HOST, DB_USERNAME, VITE_SYNCFUSION_LICENSE_KEY

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [ ] T008 Create Account entity class in src/main/java/com/ab/riskweb/domain/Account.java per data-model.md
- [ ] T009 [P] Create AccountChangeRequest entity class in src/main/java/com/ab/riskweb/domain/AccountChangeRequest.java
- [ ] T010 [P] Create ActiveStatus enum in src/main/java/com/ab/riskweb/domain/ActiveStatus.java
- [ ] T011 [P] Create OperationType enum in src/main/java/com/ab/riskweb/domain/OperationType.java
- [ ] T012 [P] Create ChangeRequestStatus enum in src/main/java/com/ab/riskweb/domain/ChangeRequestStatus.java with state transition validation
- [ ] T013 [P] Create JsonNodeConverter class in src/main/java/com/ab/riskweb/domain/JsonNodeConverter.java for JSON snapshot storage
- [ ] T014 Create AccountRepository interface in src/main/java/com/ab/riskweb/repository/AccountRepository.java with findByActiveStatus and searchActive queries
- [ ] T015 [P] Create AccountChangeRequestRepository interface in src/main/java/com/ab/riskweb/repository/AccountChangeRequestRepository.java
- [ ] T016 Create SpringSecurityKerberosConfig in src/main/java/com/ab/riskweb/config/SecurityConfig.java with SpnegoAuthenticationProcessingFilter
- [ ] T017 [P] Create JpaConfig in src/main/java/com/ab/riskweb/config/JpaConfig.java with ddl-auto=validate enforcement
- [ ] T018 [P] Create SpaFallbackController in src/main/java/com/ab/riskweb/controller/SpaFallbackController.java for React Router support
- [ ] T019 [P] Create custom exception classes (ForbiddenException, NotFoundException, ConflictException, ValidationException) in src/main/java/com/ab/riskweb/exception/
- [ ] T020 [P] Create GlobalExceptionHandler with @ControllerAdvice in src/main/java/com/ab/riskweb/exception/GlobalExceptionHandler.java for consistent error responses
- [ ] T021 [P] Create ApiResponse and ApiError DTOs in src/main/java/com/ab/riskweb/dto/ for consistent envelope format
- [ ] T022 [P] Create AB theme file in frontend/src/theme/abTheme.ts with MUI palette configuration (AB Blue, Gold, Purple, Teal, Black, White)
- [ ] T023 [P] Create Syncfusion CSS overrides in frontend/src/theme/syncfusionOverrides.css for AB Blue accents
- [ ] T024 [P] Register Syncfusion license in frontend/src/main.tsx with fail-fast on missing VITE_SYNCFUSION_LICENSE_KEY
- [ ] T025 [P] Add AB logo assets (logo-1.png, logo-2.png) to frontend/src/assets/
- [ ] T026 [P] Create TypeScript interfaces for Account and AccountChangeRequest in frontend/src/types/account.ts
- [ ] T027 [P] Create Axios client with /api/v1 base URL and error interceptor in frontend/src/api/client.ts
- [ ] T028 [P] Create navigation configuration file in frontend/src/config/navigation.ts with all 6 sections and placeholder routes

**Checkpoint**: Foundation ready - user story implementation can now begin in parallel

---

## Phase 3: User Story 10 - Navigation Shell with Placeholder Pages (Priority: P1) 🎯 MVP Foundation

**Goal**: Establish the complete AllianceBernstein-branded application shell (top nav, sidebar, main content area) with all sections visible but only Accounts functional. This is the structural foundation for the entire application.

**Independent Test**: Load the application, click each top nav section (Liquidity, SEC, Lux, OpsRisk, RiskIT, Misc), verify shell structure matches constitution screenshot exactly, verify placeholder pages render correctly for all non-Accounts routes.

### Implementation for User Story 10

- [ ] T029 [P] [US10] Create AppShell layout component in frontend/src/components/AppShell.tsx with three-zone structure (top nav, sidebar, main content)
- [ ] T030 [P] [US10] Create TopNavBar component in frontend/src/components/TopNavBar.tsx with AB logo, section links, Windows login chip display
- [ ] T031 [P] [US10] Create LeftSidebar component in frontend/src/components/LeftSidebar.tsx with collapsible navigation tree and toggle button
- [ ] T032 [P] [US10] Create WorkInProgress placeholder component in frontend/src/components/WorkInProgress.tsx with centered "Work In Progress…" text
- [ ] T033 [US10] Create React Router configuration in frontend/src/App.tsx with all section routes and WorkInProgress as default component
- [ ] T034 [US10] Implement sidebar collapse/expand animation (~200ms ease transition between 48px and 200px width)
- [ ] T035 [US10] Implement active section highlighting logic (Gold pill background, Black text) in TopNavBar component
- [ ] T036 [US10] Implement active child link highlighting logic (Gold underline or bold White text) in LeftSidebar component
- [ ] T037 [US10] Add default route redirect from / to /liquidity/overview in React Router configuration

**Checkpoint**: At this point, the navigation shell should be fully functional with all sections accessible and placeholder pages rendering correctly. Ready to add authentication and data features.

---

## Phase 4: User Story 1 - Transparent Windows SSO Authentication (Priority: P1) 🎯 MVP

**Goal**: Enable automatic authentication via Windows SSO (Kerberos/SPNEGO) without any login screen, capturing and displaying the Windows principal in the top navigation bar.

**Independent Test**: Open the application in a corporate intranet browser, verify Windows principal is captured without login form, verify principal displays as DOMAIN\username in top-right chip.

### Unit Tests for User Story 1 ⚠️

> **NOTE: Write these tests FIRST, ensure they FAIL before implementation**

- [ ] T038 [P] [US1] Create SecurityConfig test in src/test/java/com/ab/riskweb/config/SecurityConfigTest.java to verify Kerberos filter chain setup
- [ ] T039 [P] [US1] Create AuthenticationController test in src/test/java/com/ab/riskweb/controller/AuthenticationControllerTest.java to verify /api/v1/auth/me endpoint

### Implementation for User Story 1

- [ ] T040 [US1] Configure SpnegoAuthenticationProcessingFilter with keytab location and service principal in SecurityConfig.java
- [ ] T041 [US1] Configure KerberosServiceAuthenticationProvider with SunJaasKerberosTicketValidator in SecurityConfig.java
- [ ] T042 [US1] Create local development profile override in application.yml to bypass Kerberos with mock user (LOCAL\developer)
- [ ] T043 [US1] Create AuthenticationController in src/main/java/com/ab/riskweb/controller/AuthenticationController.java with GET /api/v1/auth/me endpoint
- [ ] T044 [US1] Create AuthenticationDTO in src/main/java/com/ab/riskweb/dto/AuthenticationDTO.java with principal, domain, username fields
- [ ] T045 [US1] Create AuthContext provider in frontend/src/context/AuthContext.tsx to fetch and store authenticated user principal
- [ ] T046 [US1] Create useAuth hook in frontend/src/hooks/useAuth.ts to access AuthContext
- [ ] T047 [US1] Create authApi.ts in frontend/src/api/authApi.ts with fetchCurrentUser function calling GET /api/v1/auth/me
- [ ] T048 [US1] Update TopNavBar component to display Windows login chip using useAuth hook
- [ ] T049 [US1] Implement 401 Unauthorized error handling in Axios interceptor with "Your session has expired" message
- [ ] T050 [US1] Add HTTP 401 error page component in frontend/src/components/ErrorPage.tsx for Kerberos negotiation failures

**Checkpoint**: At this point, User Story 1 should be fully functional - users are automatically authenticated via SSO and see their Windows login in the UI. Ready to add data features.

---

## Phase 5: User Story 2 - View Paginated, Searchable Account List (Priority: P1) 🎯 MVP

**Goal**: Display a paginated, searchable list of all active Fund Accounts using Syncfusion Grid with real-time filtering and sorting capabilities.

**Independent Test**: Navigate to /misc/accounts, verify Syncfusion Grid displays active accounts, verify search/filter works, verify pagination controls are functional.

### Unit Tests for User Story 2 ⚠️

- [ ] T051 [P] [US2] Create AccountRepository test in src/test/java/com/ab/riskweb/repository/AccountRepositoryTest.java to verify findByActiveStatus and searchActive queries
- [ ] T052 [P] [US2] Create AccountController test in src/test/java/com/ab/riskweb/controller/AccountControllerTest.java to verify GET /api/v1/accounts endpoint with pagination

### Implementation for User Story 2

- [ ] T053 [P] [US2] Create AccountDTO in src/main/java/com/ab/riskweb/dto/AccountDTO.java mapping all Account entity fields
- [ ] T054 [P] [US2] Create PagedResponseDTO in src/main/java/com/ab/riskweb/dto/PagedResponseDTO.java for paginated list responses
- [ ] T055 [US2] Create AccountService in src/main/java/com/ab/riskweb/service/AccountService.java with findAllActive and searchAccounts methods
- [ ] T056 [US2] Create AccountController in src/main/java/com/ab/riskweb/controller/AccountController.java with GET /api/v1/accounts endpoint (pagination, search, sort params)
- [ ] T057 [US2] Create GET /api/v1/accounts/{fundId} endpoint in AccountController for single account retrieval
- [ ] T058 [US2] Create accountsApi.ts in frontend/src/api/accountsApi.ts with fetchAccounts and fetchAccount functions
- [ ] T059 [US2] Create AccountsPage component in frontend/src/pages/accounts/AccountsPage.tsx with Syncfusion GridComponent
- [ ] T060 [US2] Configure Syncfusion Grid columns (Fund ID, Fund Name, ePace ID, Short Name, Fund Type, Reporting Frequency, ESMA/40ACT, RM Acronym, Active Status)
- [ ] T061 [US2] Implement search box with real-time filtering in AccountsPage component
- [ ] T062 [US2] Implement pagination controls (10-50 rows per page, configurable) in Syncfusion Grid
- [ ] T063 [US2] Implement column header sorting (ascending/descending toggle) in Syncfusion Grid
- [ ] T064 [US2] Apply AB Blue row selection highlight (rgba(30,155,215,0.15)) to Syncfusion Grid via CSS
- [ ] T065 [US2] Update React Router in App.tsx to map /misc/accounts route to AccountsPage component (replace WorkInProgress)

**Checkpoint**: At this point, User Story 2 should be fully functional - users can view, search, filter, and sort active accounts. Ready to add maker-checker workflow.

---

## Phase 6: User Story 7 - View All Pending Change Requests (Checker Queue) (Priority: P1) 🎯 MVP

**Goal**: Display all pending change requests submitted by other users, enabling the checker role to review and action requests.

**Independent Test**: Navigate to /misc/accounts/pending, verify all PENDING_APPROVAL requests from other users are displayed, verify approve/reject buttons are disabled for own requests.

### Unit Tests for User Story 7 ⚠️

- [ ] T066 [P] [US7] Create AccountChangeRequestRepository test in src/test/java/com/ab/riskweb/repository/AccountChangeRequestRepositoryTest.java to verify findByStatus and findConflictingRequests queries
- [ ] T067 [P] [US7] Create AccountChangeRequestController test in src/test/java/com/ab/riskweb/controller/AccountChangeRequestControllerTest.java to verify GET /api/v1/accounts/requests endpoint

### Implementation for User Story 7

- [ ] T068 [P] [US7] Create AccountChangeRequestDTO in src/main/java/com/ab/riskweb/dto/AccountChangeRequestDTO.java with all fields including hasConflicts and conflictCount
- [ ] T069 [P] [US7] Create AccountChangeRequestListDTO in src/main/java/com/ab/riskweb/dto/AccountChangeRequestListDTO.java with preview fields (fundIdPreview, fundNamePreview)
- [ ] T070 [US7] Create AccountChangeRequestService in src/main/java/com/ab/riskweb/service/AccountChangeRequestService.java with findPendingRequests and findConflictingRequests methods
- [ ] T071 [US7] Create AccountChangeRequestController in src/main/java/com/ab/riskweb/controller/AccountChangeRequestController.java with GET /api/v1/accounts/requests endpoint
- [ ] T072 [US7] Create GET /api/v1/accounts/requests/{requestId} endpoint in AccountChangeRequestController for request detail retrieval
- [ ] T073 [US7] Create changeRequestsApi.ts in frontend/src/api/changeRequestsApi.ts with fetchPendingRequests and fetchRequestDetail functions
- [ ] T074 [US7] Create PendingApprovalsPage component in frontend/src/pages/accounts/PendingApprovalsPage.tsx with Syncfusion GridComponent
- [ ] T075 [US7] Configure Syncfusion Grid columns (Request ID, Operation, Fund ID, Fund Name, Maker, Submitted On, Actions)
- [ ] T076 [US7] Implement operation type badge rendering (Gold for CREATE, Purple for UPDATE, Teal for DELETE) in grid cell template
- [ ] T077 [US7] Implement approve/reject button logic with disabled state for own requests (makerLogin === current user)
- [ ] T078 [US7] Implement tooltip display "You cannot approve your own request" on disabled approve/reject buttons
- [ ] T079 [US7] Implement conflict warning badge display when hasConflicts === true ("Warning: N other pending request(s) exist")
- [ ] T080 [US7] Update navigation.ts to add "Pending Approvals" link under Misc → Accounts
- [ ] T081 [US7] Update React Router in App.tsx to map /misc/accounts/pending route to PendingApprovalsPage component

**Checkpoint**: At this point, User Story 7 should be fully functional - checkers can view all pending requests with conflict warnings and maker-checker UI enforcement. Ready to add approval actions.

---

## Phase 7: User Story 8 - Approve Pending Change Request (Priority: P1) 🎯 MVP

**Goal**: Enable checkers to approve pending change requests, causing the live Account record to be atomically updated within a transaction.

**Independent Test**: Approve a CREATE request, verify new account appears in live account list; approve an UPDATE request, verify live record is updated; approve a DELETE request, verify account becomes INACTIVE.

### Unit Tests for User Story 8 ⚠️ CRITICAL

> **MANDATORY TEST**: This test MUST pass. If removed or broken, the build should fail.

- [ ] T082 [US8] Create AccountChangeRequestService test in src/test/java/com/ab/riskweb/service/AccountChangeRequestServiceTest.java
- [ ] T083 [US8] Add test_approveOwnRequest_shouldThrowForbidden test to verify ForbiddenException when maker === checker
- [ ] T084 [US8] Add test_approveCreate_shouldInsertAccount test to verify CREATE approval inserts new Account record
- [ ] T085 [US8] Add test_approveUpdate_shouldUpdateAccount test to verify UPDATE approval updates live Account record
- [ ] T086 [US8] Add test_approveDelete_shouldSoftDeleteAccount test to verify DELETE approval sets activeStatus to INACTIVE
- [ ] T087 [US8] Add test_approveAlreadyApproved_shouldThrowConflict test to verify ConflictException when status is not PENDING_APPROVAL

### Implementation for User Story 8

- [ ] T088 [US8] Implement approve method in AccountChangeRequestService.java with maker ≠ checker validation (throw ForbiddenException if equal)
- [ ] T089 [US8] Implement state transition validation in approve method (throw ConflictException if status != PENDING_APPROVAL)
- [ ] T090 [US8] Implement CREATE approval logic: insert new Account record with createdBy = makerLogin, createdOn = approval timestamp
- [ ] T091 [US8] Implement UPDATE approval logic: update live Account record fields from snapshot, preserve createdBy/createdOn
- [ ] T092 [US8] Implement DELETE approval logic: set Account.activeStatus = INACTIVE (soft delete)
- [ ] T093 [US8] Ensure atomic transaction: update AccountChangeRequest status to APPROVED and live Account in same @Transactional method
- [ ] T094 [US8] Set checkerLogin and checkerTimestamp fields on AccountChangeRequest record during approval
- [ ] T095 [US8] Create POST /api/v1/accounts/requests/{requestId}/approve endpoint in AccountChangeRequestController
- [ ] T096 [US8] Create approveRequest function in changeRequestsApi.ts calling POST /api/v1/accounts/requests/{requestId}/approve
- [ ] T097 [US8] Implement approve button click handler in PendingApprovalsPage component with confirmation dialog
- [ ] T098 [US8] Implement success message display (MUI Snackbar or Syncfusion Toast) after successful approval
- [ ] T099 [US8] Implement error handling for HTTP 403 FORBIDDEN (self-approval attempt) with user-friendly message
- [ ] T100 [US8] Implement error handling for HTTP 409 CONFLICT (already approved) with user-friendly message
- [ ] T101 [US8] Refresh pending requests grid after successful approval to reflect updated state

**Checkpoint**: At this point, User Story 8 should be fully functional - checkers can approve requests with full maker-checker enforcement, and live accounts are updated atomically. Core workflow is complete.

---

## Phase 8: User Story 3 - Submit New Fund Account for Approval (Priority: P1) 🎯 MVP

**Goal**: Enable makers to submit new Fund Account creation requests through a Syncfusion form modal with validation.

**Independent Test**: Click "Create Account", fill form, submit, verify pending AccountChangeRequest record appears in checker queue, verify live account list is unchanged.

### Unit Tests for User Story 3 ⚠️

- [ ] T102 [P] [US3] Create AccountChangeRequestController test to verify POST /api/v1/accounts/requests endpoint with CREATE operation
- [ ] T103 [P] [US3] Create validation tests to verify required fields (fundId, fundName, fundType) enforcement
- [ ] T103a [P] [US3] Create AccountFormInput validation test in src/test/java/com/ab/riskweb/dto/AccountFormInputTest.java to verify @NotNull constraints fail when fundId, fundName, or fundType are null/blank

### Implementation for User Story 3

- [ ] T104 [P] [US3] Create AccountFormInput DTO in src/main/java/com/ab/riskweb/dto/AccountFormInput.java with validation annotations (@NotNull on fundId, fundName, fundType)
- [ ] T105 [P] [US3] Create CreateAccountRequest DTO in src/main/java/com/ab/riskweb/dto/CreateAccountRequest.java wrapping operationType and accountData
- [ ] T106 [US3] Implement createChangeRequest method in AccountChangeRequestService.java to create PENDING_APPROVAL request with JSON snapshot
- [ ] T107 [US3] Create POST /api/v1/accounts/requests endpoint in AccountChangeRequestController for CREATE operations
- [ ] T108 [US3] Capture makerLogin from Spring Security context and set makerTimestamp to current UTC time in createChangeRequest method
- [ ] T109 [US3] Create createAccountRequest function in changeRequestsApi.ts calling POST /api/v1/accounts/requests
- [ ] T110 [US3] Create AccountFormDialog component in frontend/src/components/accounts/AccountFormDialog.tsx with Syncfusion form inputs
- [ ] T111 [US3] Configure form fields: fundId (TextBoxComponent), fundName (TextBoxComponent), epaceId (TextBoxComponent), shortName (TextBoxComponent), fundType (DropDownListComponent), reportingFrequency (DropDownListComponent), esma40Act (DropDownListComponent), rmAcronym (TextBoxComponent)
- [ ] T112 [US3] Implement required field validation for fundId, fundName, fundType with inline error messages (White text)
- [ ] T113 [US3] Add "Create Account" button (AB Blue primary, White text) above Syncfusion Grid in AccountsPage component
- [ ] T114 [US3] Implement "Create Account" button click handler to open AccountFormDialog modal
- [ ] T115 [US3] Implement "Submit for Approval" button in AccountFormDialog calling createAccountRequest API
- [ ] T116 [US3] Implement success message display after submission ("Account creation request submitted for approval")
- [ ] T117 [US3] Implement Cancel button in AccountFormDialog to close modal and discard unsaved data
- [ ] T118 [US3] Ensure form modal has Black background with White labels per AB brand guidelines

**Checkpoint**: At this point, User Story 3 should be fully functional - makers can submit new account creation requests, and the requests appear in the checker queue for approval.

---

## Phase 9: User Story 4 - Submit Edit to Existing Account for Approval (Priority: P2)

**Goal**: Enable makers to submit updates to existing Fund Accounts through a pre-populated form modal, with changes tracked in the snapshot.

**Independent Test**: Select an account, click "Edit", modify fields, submit, verify live record remains unchanged until approval, verify snapshot captures only changed fields.

### Unit Tests for User Story 4 ⚠️

- [ ] T119 [P] [US4] Create AccountChangeRequestController test to verify PUT /api/v1/accounts/requests/{requestId} endpoint with UPDATE operation
- [ ] T120 [P] [US4] Create snapshot diff test to verify UPDATE snapshot includes "from" and "to" values for changed fields only

### Implementation for User Story 4

- [ ] T121 [P] [US4] Create UpdateAccountRequest DTO in src/main/java/com/ab/riskweb/dto/UpdateAccountRequest.java with operationType and accountData fields
- [ ] T122 [US4] Implement createUpdateChangeRequest method in AccountChangeRequestService.java to generate diff snapshot (fields with "from" and "to" values)
- [ ] T123 [US4] Create PUT /api/v1/accounts/requests/{requestId} endpoint in AccountChangeRequestController for UPDATE operations
- [ ] T124 [US4] Create updateAccountRequest function in changeRequestsApi.ts calling PUT /api/v1/accounts/requests/{requestId}
- [ ] T125 [US4] Add "Edit" action button (icon button or context menu) to each grid row in AccountsPage component
- [ ] T126 [US4] Implement "Edit" button click handler to open AccountFormDialog pre-populated with current account values
- [ ] T127 [US4] Implement field change tracking in AccountFormDialog to highlight modified fields with light Gold border (optional visual indicator)
- [ ] T128 [US4] Implement "Submit for Approval" button for edit mode calling updateAccountRequest API with only changed fields
- [ ] T129 [US4] Implement success message display after submission with note that live record is unchanged until approval

**Checkpoint**: At this point, User Story 4 should be fully functional - makers can submit account edit requests with field-level change tracking, and the requests appear in the checker queue.

---

## Phase 10: User Story 5 - Submit Deactivation (Soft-Delete) for Approval (Priority: P2)

**Goal**: Enable makers to submit deactivation requests (soft-delete) for active Fund Accounts through a confirmation dialog.

**Independent Test**: Click "Deactivate" on an active account, confirm, verify live record remains ACTIVE until approval, verify DELETE request appears in checker queue.

### Unit Tests for User Story 5 ⚠️

- [ ] T130 [P] [US5] Create AccountChangeRequestController test to verify DELETE /api/v1/accounts/requests/{requestId} endpoint with DELETE operation
- [ ] T131 [P] [US5] Create AccountChangeRequestService test to verify DELETE approval sets activeStatus to INACTIVE without hard delete

### Implementation for User Story 5

- [ ] T132 [P] [US5] Create DeleteAccountRequest DTO in src/main/java/com/ab/riskweb/dto/DeleteAccountRequest.java with operationType and accountId fields
- [ ] T133 [US5] Implement createDeleteChangeRequest method in AccountChangeRequestService.java to capture current account state in snapshot
- [ ] T134 [US5] Create DELETE /api/v1/accounts/requests/{requestId} endpoint in AccountChangeRequestController for DELETE operations
- [ ] T135 [US5] Create deleteAccountRequest function in changeRequestsApi.ts calling DELETE /api/v1/accounts/requests/{requestId}
- [ ] T136 [US5] Add "Deactivate" action button (icon button with MUI default error red, not AB accent colors) to each grid row in AccountsPage component
- [ ] T137 [US5] Implement "Deactivate" button click handler to open confirmation dialog with warning message
- [ ] T138 [US5] Create DeactivateConfirmDialog component in frontend/src/components/accounts/DeactivateConfirmDialog.tsx
- [ ] T139 [US5] Implement "Submit for Approval" button (error red background, White text) in confirmation dialog
- [ ] T140 [US5] Implement Cancel button (secondary outlined) in confirmation dialog
- [ ] T141 [US5] Implement success message display after submission confirming deactivation request is pending approval
- [ ] T142 [US5] Add filter toggle or checkbox to AccountsPage grid to show inactive accounts when explicitly requested (includeInactive query param)

**Checkpoint**: At this point, User Story 5 should be fully functional - makers can submit deactivation requests, and the requests appear in the checker queue for approval.

---

## Phase 11: User Story 9 - Reject Pending Change Request with Reason (Priority: P2)

**Goal**: Enable checkers to reject pending change requests with mandatory rejection reason, allowing makers to review feedback and resubmit.

**Independent Test**: Click "Reject" on a pending request, enter reason, verify request transitions to REJECTED, verify reason is stored and displayed to maker.

### Unit Tests for User Story 9 ⚠️

- [ ] T143 [P] [US9] Create AccountChangeRequestService test to verify reject method enforces maker ≠ checker
- [ ] T144 [P] [US9] Create validation test to verify rejection reason is required (throws ValidationException if null/empty)

### Implementation for User Story 9

- [ ] T145 [P] [US9] Create RejectRequestDTO in src/main/java/com/ab/riskweb/dto/RejectRequestDTO.java with reason field (@NotNull, @NotBlank validation)
- [ ] T146 [US9] Implement reject method in AccountChangeRequestService.java with maker ≠ checker validation and state transition checks
- [ ] T147 [US9] Set rejectionReason, checkerLogin, and checkerTimestamp fields on AccountChangeRequest record during rejection
- [ ] T148 [US9] Create POST /api/v1/accounts/requests/{requestId}/reject endpoint in AccountChangeRequestController
- [ ] T149 [US9] Create rejectRequest function in changeRequestsApi.ts calling POST /api/v1/accounts/requests/{requestId}/reject
- [ ] T150 [US9] Create RejectionDialog component in frontend/src/components/accounts/RejectionDialog.tsx with text area for reason (required field)
- [ ] T151 [US9] Implement "Reject" button click handler in PendingApprovalsPage to open RejectionDialog
- [ ] T152 [US9] Implement "Reject Request" submit button (error red primary) in RejectionDialog calling rejectRequest API
- [ ] T153 [US9] Implement inline validation error display if reason field is blank ("Rejection reason is required" in White text)
- [ ] T154 [US9] Implement success message display after successful rejection
- [ ] T155 [US9] Refresh pending requests grid after successful rejection to reflect updated state
- [ ] T156 [US9] Add rejection reason display to AccountChangeRequestDTO and show in request detail view for makers

**Checkpoint**: At this point, User Story 9 should be fully functional - checkers can reject requests with mandatory reasons, and makers can view the feedback.

---

## Phase 12: User Story 6 - Cancel Own Pending Change Request (Priority: P3)

**Goal**: Enable makers to cancel their own pending change requests before approval, removing them from the checker queue.

**Independent Test**: Submit a change request, navigate to "My Pending Requests", click "Cancel", verify request transitions to CANCELLED and disappears from both maker and checker queues.

### Unit Tests for User Story 6 ⚠️

- [ ] T157 [P] [US6] Create AccountChangeRequestService test to verify cancel method enforces makerLogin === current user
- [ ] T158 [P] [US6] Create test to verify cancel fails with ForbiddenException if user is not the original maker

### Implementation for User Story 6

- [ ] T159 [US6] Implement cancel method in AccountChangeRequestService.java with makerLogin === current user validation
- [ ] T160 [US6] Ensure cancel method only allows PENDING_APPROVAL → CANCELLED state transition (throw ConflictException if already approved/rejected)
- [ ] T161 [US6] Create DELETE /api/v1/accounts/requests/{requestId}/cancel endpoint in AccountChangeRequestController
- [ ] T162 [US6] Create cancelRequest function in changeRequestsApi.ts calling DELETE /api/v1/accounts/requests/{requestId}/cancel
- [ ] T163 [US6] Create MyPendingRequestsPage component in frontend/src/pages/accounts/MyPendingRequestsPage.tsx (or add filter toggle to PendingApprovalsPage)
- [ ] T164 [US6] Implement "Cancel Request" button (secondary outlined White border) next to each of user's own pending requests
- [ ] T165 [US6] Implement "Cancel Request" button click handler with optional confirmation dialog ("Are you sure you want to cancel this request?")
- [ ] T166 [US6] Implement success message display after successful cancellation
- [ ] T167 [US6] Refresh pending requests grid after successful cancellation to reflect updated state
- [ ] T168 [US6] Update navigation.ts to add "My Pending Requests" link under Misc → Accounts (optional sub-page)
- [ ] T169 [US6] Update React Router in App.tsx to map /misc/accounts/my-pending route to MyPendingRequestsPage component

**Checkpoint**: At this point, User Story 6 should be fully functional - makers can cancel their own pending requests, reducing unnecessary checker workload.

---

## Phase 13: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories

- [ ] T170 [P] Add SLF4J logging for all service methods (maker-checker actions, approval/rejection, errors) in AccountService and AccountChangeRequestService
- [ ] T171 [P] Add Javadoc comments to all public service methods and controllers per Constitution Article VIII
- [ ] T172 [P] Create inter font import in frontend/src/index.css with fallback to Helvetica Neue, Arial
- [ ] T172a [P] Verify error state components (Deactivate button T136, Reject button T151) use MUI default error palette (#d32f2f or theme.palette.error.main) not AB accent colors, per Constitution Article X.2 usage rules
- [ ] T173 [P] Add NotificationContext provider in frontend/src/context/NotificationContext.tsx for consistent success/error message display across components
- [ ] T174 [P] Implement loading spinner component in frontend/src/components/LoadingSpinner.tsx with AB Blue accent
- [ ] T175 [P] Add Spring Boot Actuator health endpoint configuration with /actuator/health unauthenticated for load balancer probes
- [ ] T176 [P] Configure HikariCP connection pool settings (maximum-pool-size: 10, minimum-idle: 5) in application.yml per research.md "Database Connection Pooling" section
- [ ] T177 [P] Add CORS configuration in SecurityConfig.java for local development (disabled in production)
- [ ] T178 [P] Add application version and build timestamp to top navigation bar or footer
- [ ] T179 [P] Implement responsive design adjustments for sidebar collapse on narrow screens (<1024px width)
- [ ] T180 [P] Add frontend error boundary component in frontend/src/components/ErrorBoundary.tsx to catch React rendering errors
- [ ] T181 [P] Configure Maven build profile for production with minification and source map generation
- [ ] T182 [P] Add sample data SQL script in docs/sample-data.sql for development environment setup
- [ ] T183 [P] Update CLAUDE.md with project commands and code style guidelines (auto-generated from plan.md)
- [ ] T184 Run full backend test suite with coverage report (mvn test jacoco:report) and verify >80% coverage for service layer
- [ ] T185 Validate quickstart.md instructions by following setup steps in a clean environment
- [ ] T186 Build production JAR (mvn clean package -Pprod) and verify React build is embedded in /src/main/resources/static/
- [ ] T187 Start production JAR and verify health endpoint returns HTTP 200 with status UP
- [ ] T188 Smoke test all user stories end-to-end in local environment with mock Kerberos user

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS all user stories
- **User Story 10 (Phase 3)**: Depends on Foundational completion - BLOCKS all other user stories (provides navigation shell)
- **User Story 1 (Phase 4)**: Depends on US10 completion - BLOCKS all data-driven user stories (provides authentication)
- **User Stories 2, 7 (Phase 5-6)**: Depend on US1 completion - can proceed in parallel
- **User Story 8 (Phase 7)**: Depends on US7 completion - BLOCKS maker workflow stories (provides approval mechanism)
- **User Stories 3, 4, 5 (Phase 8-10)**: Depend on US8 completion - can proceed in parallel
- **User Stories 9, 6 (Phase 11-12)**: Depend on US8 completion - can proceed in parallel with US3-5
- **Polish (Phase 13)**: Depends on all desired user stories being complete

### User Story Dependencies

- **User Story 10 (P1)**: Navigation Shell - No dependencies on other stories (depends on Foundational only) ✅ START HERE
- **User Story 1 (P1)**: Authentication - Depends on US10 (needs shell to display login chip) ✅ THEN THIS
- **User Story 2 (P1)**: View Accounts - Depends on US1 (needs authentication) ✅ THEN THIS
- **User Story 7 (P1)**: View Pending Requests - Depends on US1 (needs authentication) ✅ CAN RUN IN PARALLEL WITH US2
- **User Story 8 (P1)**: Approve Requests - Depends on US7 (needs checker queue UI) ✅ THEN THIS
- **User Story 3 (P1)**: Create Account - Depends on US8 (needs approval workflow) ✅ THEN THIS
- **User Story 4 (P2)**: Edit Account - Depends on US2 and US8 (needs account list and approval workflow) ✅ CAN RUN IN PARALLEL WITH US3
- **User Story 5 (P2)**: Delete Account - Depends on US2 and US8 (needs account list and approval workflow) ✅ CAN RUN IN PARALLEL WITH US3-4
- **User Story 9 (P2)**: Reject Requests - Depends on US7 (needs checker queue UI) ✅ CAN RUN IN PARALLEL WITH US3-5
- **User Story 6 (P3)**: Cancel Own Requests - Depends on US3 (needs maker workflow UI) ✅ CAN RUN IN PARALLEL WITH US3-5, US9

### Within Each User Story

- Tests MUST be written and FAIL before implementation
- Entities/DTOs before services
- Services before controllers
- API endpoints before frontend API client functions
- Frontend API client before UI components
- UI components before route mapping
- Story complete before moving to next priority

### Parallel Opportunities

**Phase 1 - Setup**: All tasks marked [P] can run in parallel (T002-T007)

**Phase 2 - Foundational**: All tasks marked [P] can run in parallel within their categories:
- Entities: T009-T013 in parallel
- Repositories: T015 in parallel with entities
- Config: T016-T018 in parallel
- Exceptions/DTOs: T019-T021 in parallel
- Frontend theme: T022-T028 in parallel

**Phase 3 - US10**: All tasks marked [P] can run in parallel (T029-T032)

**Phase 4 - US1**: Test tasks T038-T039 in parallel, implementation tasks T040-T042 in parallel, frontend tasks T045-T047 in parallel

**Phase 5 - US2**: Test tasks T051-T052 in parallel, DTO tasks T053-T054 in parallel, API functions T058 in parallel with UI components T059-T064

**Multiple user stories can run in parallel once their dependencies are met** (e.g., US3, US4, US5, US9 can all run simultaneously after US8 is complete)

---

## Parallel Example: User Story 8 (Approve Requests)

```bash
# Launch all unit tests for User Story 8 together:
Task: "Create AccountChangeRequestService test in src/test/java/.../AccountChangeRequestServiceTest.java"
Task: "Add test_approveOwnRequest_shouldThrowForbidden test" (same file, different methods)
Task: "Add test_approveCreate_shouldInsertAccount test"

# After tests fail, launch implementation tasks:
Task: "Implement approve method with maker ≠ checker validation"
Task: "Create POST /api/v1/accounts/requests/{requestId}/approve endpoint"
Task: "Create approveRequest function in changeRequestsApi.ts"
```

---

## Implementation Strategy

### MVP First (Minimum Viable Product)

Complete these phases to achieve a functional maker-checker workflow:

1. **Phase 1**: Setup → Project structure ready
2. **Phase 2**: Foundational → Core entities and infrastructure ready ⚠️ BLOCKS ALL STORIES
3. **Phase 3**: User Story 10 → Navigation shell ready ⚠️ BLOCKS ALL STORIES
4. **Phase 4**: User Story 1 → Authentication ready ⚠️ BLOCKS ALL STORIES
5. **Phase 5**: User Story 2 → Account viewing ready
6. **Phase 6**: User Story 7 → Checker queue ready
7. **Phase 7**: User Story 8 → Approval workflow ready ✅ CORE WORKFLOW COMPLETE
8. **Phase 8**: User Story 3 → Account creation ready ✅ MVP COMPLETE - STOP and VALIDATE

**STOP and VALIDATE**: Test the complete maker-checker workflow end-to-end:
- Authenticate via SSO
- View account list
- Create new account request
- View pending request in checker queue
- Approve request
- Verify new account appears in list
- Verify maker-checker enforcement (cannot approve own request)

### Incremental Delivery

After MVP validation, add features incrementally:

1. **MVP (US10 + US1 + US2 + US7 + US8 + US3)** → Deploy/Demo ✅
2. Add **US4 (Edit Account)** → Test independently → Deploy/Demo
3. Add **US5 (Delete Account)** → Test independently → Deploy/Demo
4. Add **US9 (Reject Requests)** → Test independently → Deploy/Demo
5. Add **US6 (Cancel Requests)** → Test independently → Deploy/Demo
6. Add **Phase 13 (Polish)** → Final production-ready deployment

Each increment adds value without breaking previous functionality.

### Parallel Team Strategy

With multiple developers:

1. **All developers**: Complete Phase 1 (Setup) and Phase 2 (Foundational) together ⚠️ CRITICAL PATH
2. **Developer A**: Phase 3 (US10 - Navigation Shell) ⚠️ BLOCKS OTHERS
3. **Developer B**: Phase 4 (US1 - Authentication) after US10 complete ⚠️ BLOCKS OTHERS
4. **After US1 complete**:
   - **Developer A**: Phase 5 (US2 - View Accounts)
   - **Developer B**: Phase 6 (US7 - Checker Queue)
5. **Developer C**: Phase 7 (US8 - Approve Requests) after US7 complete ⚠️ BLOCKS MAKER WORKFLOWS
6. **After US8 complete** (all developers can work in parallel):
   - **Developer A**: Phase 8 (US3 - Create Account)
   - **Developer B**: Phase 9 (US4 - Edit Account)
   - **Developer C**: Phase 10 (US5 - Delete Account)
   - **Developer D**: Phase 11 (US9 - Reject Requests)
   - **Developer E**: Phase 12 (US6 - Cancel Requests)
7. **All developers**: Phase 13 (Polish) together

---

## Summary

- **Total Tasks**: 188
- **Task Count by Phase**:
  - Phase 1 (Setup): 7 tasks
  - Phase 2 (Foundational): 21 tasks
  - Phase 3 (US10 - Navigation Shell): 9 tasks
  - Phase 4 (US1 - Authentication): 13 tasks
  - Phase 5 (US2 - View Accounts): 15 tasks
  - Phase 6 (US7 - Checker Queue): 16 tasks
  - Phase 7 (US8 - Approve Requests): 20 tasks (includes 6 mandatory unit tests)
  - Phase 8 (US3 - Create Account): 17 tasks
  - Phase 9 (US4 - Edit Account): 11 tasks
  - Phase 10 (US5 - Delete Account): 11 tasks
  - Phase 11 (US9 - Reject Requests): 14 tasks
  - Phase 12 (US6 - Cancel Requests): 11 tasks
  - Phase 13 (Polish): 19 tasks

- **Parallel Opportunities**: 63 tasks marked [P] can run in parallel within their phase
- **MVP Scope**: Phases 1-8 (138 tasks) deliver a complete, testable maker-checker workflow
- **Independent Test Criteria**: Each user story phase includes clear acceptance criteria and checkpoint for validation
- **Critical Path**: Setup → Foundational → US10 → US1 → US2 + US7 → US8 → Maker Workflows (US3-5, US9, US6)

---

## Format Validation

✅ All tasks follow the checklist format: `- [ ] [ID] [P?] [Story?] Description with file path`
✅ All user story tasks include [Story] label (US1, US2, etc.)
✅ All task descriptions include specific file paths
✅ All phases include checkpoint descriptions
✅ All user stories include independent test criteria
✅ All dependencies are clearly documented

---

## Notes

- **[P] tasks** = different files, no dependencies, can run in parallel
- **[Story] label** maps task to specific user story for traceability (required for US phases, omitted for Setup/Foundational/Polish)
- **Unit tests are mandatory** for backend service layer per Constitution Article IX
- **Frontend tests are optional** in this phase (not mandated by constitution)
- **Each user story is independently testable** - can stop at any checkpoint to validate story functionality
- **Maker-checker enforcement test (T083) is CRITICAL** - if removed or broken, the build should fail
- **Verify tests fail before implementing** - follow TDD practice
- **Commit after each task or logical group** for clean git history
- **Avoid**: vague tasks, same-file conflicts, cross-story dependencies that break independence
