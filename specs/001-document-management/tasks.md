---

description: "Task list for document upload and management"
---

# Tasks: Document Upload and Management

**Input**: Design documents from `/specs/001-document-management/`

**Prerequisites**: `plan.md`, `spec.md`, `research.md`, `data-model.md`, and `contracts/document-content-route.md`

**Tests**: Included because Constitution Principle V requires automated coverage for important behavior, especially authorization and data access. Write story tests before implementation and confirm they fail for the missing behavior.

**Organization**: Tasks are grouped by user story. Every task has a sequential ID and a concrete repository path.

## Format

`- [ ] T### [P] [US#] Action and repository path`

`[P]` appears only on tasks that can be done concurrently without editing the same files or depending on unfinished work. Story labels appear only in user-story phases.

## Phase 1: Setup

**Purpose**: Establish the test project and reusable test support.

- [x] T001 Create `ContosoDashboard-SSD/ContosoDashboard.Tests/ContosoDashboard.Tests.csproj` targeting `net10.0`, reference `ContosoDashboard.csproj`, and add xUnit, `Microsoft.NET.Test.Sdk`, and ASP.NET Core test-host dependencies.
- [x] T002 Add SQLite in-memory, authenticated test-host, and temporary-storage helpers in `ContosoDashboard-SSD/ContosoDashboard.Tests/Infrastructure/TestDatabase.cs`, `ContosoDashboard-SSD/ContosoDashboard.Tests/Infrastructure/ContosoWebApplicationFactory.cs`, and `ContosoDashboard-SSD/ContosoDashboard.Tests/Infrastructure/TemporaryStorageRoot.cs`.

---

## Phase 2: Foundational

**Purpose**: Protect existing SQLite data and establish infrastructure required by every story.

- [x] T003 Add a failing legacy-database integration test in `ContosoDashboard-SSD/ContosoDashboard.Tests/Integration/SqliteMigrationTests.cs` that starts from a populated `EnsureCreated` database and verifies existing users, projects, and tasks survive schema upgrade.
- [x] T004 Replace nondeterministic `DateTime.UtcNow` values in `HasData` seed definitions with stable UTC values in `ContosoDashboard-SSD/ContosoDashboard/Data/ApplicationDbContext.cs`.
- [x] T005 Generate and review an EF Core baseline migration in `ContosoDashboard-SSD/ContosoDashboard/Data/Migrations/` using `dotnet ef migrations add InitialBaseline --project ContosoDashboard-SSD/ContosoDashboard/ContosoDashboard.csproj --startup-project ContosoDashboard-SSD/ContosoDashboard/ContosoDashboard.csproj --output-dir Data/Migrations`; preserve the current schema and seed data.
- [x] T006 Implement `ContosoDashboard-SSD/ContosoDashboard/Data/SqliteMigrationBootstrapper.cs` to apply migrations to a fresh database, verify an existing `EnsureCreated` schema before recording the matching baseline, and stop with an actionable error on mismatch; never drop or recreate the database.
- [x] T007 Replace `EnsureCreated()` startup behavior in `ContosoDashboard-SSD/ContosoDashboard/Program.cs` with the safe migration bootstrapper and asynchronous migration application.
- [x] T008 Add validated `DocumentStorageOptions` and `MalwareScannerOptions` in `ContosoDashboard-SSD/ContosoDashboard/Options/DocumentStorageOptions.cs`, `ContosoDashboard-SSD/ContosoDashboard/Options/MalwareScannerOptions.cs`, and `ContosoDashboard-SSD/ContosoDashboard/appsettings.json`; keep file storage outside `wwwroot` and the ClamAV endpoint bound to loopback.
- [x] T009 Define replaceable storage and scanner contracts in `ContosoDashboard-SSD/ContosoDashboard/Services/ILocalFileStorage.cs` and `ContosoDashboard-SSD/ContosoDashboard/Services/IMalwareScanner.cs`, including asynchronous streaming, cancellation, and clean/infected/unavailable/indeterminate scan outcomes.

**Checkpoint**: Test project, migration safety, and shared dependency contracts are ready before story work.

---

## Phase 3: User Story 1 - Upload and Organize Documents (P1, MVP)

**Goal**: Authenticated employees upload supported files, receive per-file progress/results, and see accepted documents in My Documents.

**Independent Test**: Upload a clean supported file with title and category; verify the successful result and the listed title, category, date, size, project, and type. Verify size/type/scanner failures never create an available document.

### Tests for User Story 1

- [x] T010 [P] [US1] Add file validation tests in `ContosoDashboard-SSD/ContosoDashboard.Tests/Unit/DocumentFileValidationTests.cs` for PDF, DOCX, XLSX, PPTX, plain text, JPEG, and PNG signatures, rejection of DOC/XLS/PPT and mismatched signatures, and the 25 MB per-file limit.
- [x] T011 [P] [US1] Add upload workflow tests in `ContosoDashboard-SSD/ContosoDashboard.Tests/Integration/DocumentUploadTests.cs` for clean scan success, per-file results, progress reporting, scanner unavailable/timeout/indeterminate failures, and rejected-upload audit records.

### Implementation for User Story 1

- [x] T012 [US1] Create `ContosoDashboard-SSD/ContosoDashboard/Models/Document.cs` with these exact rules: `Title: Required; 1-255 characters`; `Description: Optional; maximum 2,000 characters`; `Category: Required text; one of Project Documents, Team Resources, Personal Files, Reports, Presentations, Other`; `OriginalFileName: Required display-only value; never used to form a path`; `RelativeStoragePath: Required generated path relative to the configured private storage root`; `ContentType: Required validated media type; maximum 255 characters` for PDF/DOCX/XLSX/PPTX/plain text/JPEG/PNG only; `FileSizeBytes: Required positive size; at most the configured 25 MB limit`; `UploadedByUserId: Required FK to User; the uploader and owner`; `UploadedAtUtc: Required UTC timestamp`; `ProjectId: Optional FK to Project`; `MalwareScannedAtUtc: Required timestamp set only after a clean scan`; `ScannerVersion: Optional scanner engine/signature version for diagnostic audit`.
- [x] T013 [US1] Create `ContosoDashboard-SSD/ContosoDashboard/Models/DocumentTag.cs` with `DocumentTagId: Integer primary key`, `DocumentId: Required FK to Document`, and `Value: Required, trimmed, case-normalized text; maximum 100 characters`.
- [x] T014 [US1] Create `ContosoDashboard-SSD/ContosoDashboard/Models/DocumentActivity.cs` with `DocumentActivityId: Integer primary key`, `DocumentId: Nullable integer snapshot, not a cascading FK`, `DocumentTitleSnapshot: Optional title snapshot for events after permanent deletion`, `ActorUserId: Required FK to User`, `Action: Required bounded action name`, `Outcome: Required success/rejected/failed value`, `OccurredAtUtc: Required UTC timestamp`, and `TargetSnapshot: Optional recipient or team snapshot, without file content or secrets`.
- [x] T015 [US1] Configure document/tag/activity relationships, indexes, and a migration in `ContosoDashboard-SSD/ContosoDashboard/Data/ApplicationDbContext.cs` and `ContosoDashboard-SSD/ContosoDashboard/Data/Migrations/`; enforce `(DocumentId, Value)` uniqueness, index tag `Value`, uploader+upload time, project+upload time, category+upload time, and title, and preserve activity after document deletion.
- [x] T016 [P] [US1] Implement generated-name staging, sequential streaming, finalization, compensation cleanup, and private-root path checks in `ContosoDashboard-SSD/ContosoDashboard/Services/LocalFileStorageService.cs`.
- [x] T017 [P] [US1] Implement the loopback-only ClamAV adapter in `ContosoDashboard-SSD/ContosoDashboard/Services/ClamAvMalwareScanner.cs`; enforce timeout and fail-closed handling for unavailable, infected, and indeterminate results.
- [x] T018 [US1] Implement upload validation and persistence in `ContosoDashboard-SSD/ContosoDashboard/Services/DocumentService.cs`; enforce the exact model constraints from `data-model.md`, validate signatures rather than trusting browser MIME/name, scan before availability, and audit accepted and rejected outcomes.
- [x] T019 [US1] Implement the upload form, per-file progress/results, title/category/description/project/tag inputs, and basic My Documents list in `ContosoDashboard-SSD/ContosoDashboard/Pages/Documents.razor`; cap each batch at 10 files and each file at 25 MB.

**Checkpoint**: US1 works without later stories and can be demonstrated as the MVP.

---

## Phase 4: User Story 2 - Find and Use Authorized Documents (P1)

**Goal**: Users browse, sort, filter, search, preview, and download only documents they may access.

**Independent Test**: Seed documents for multiple users and projects; verify list/search exclusion, authorization on direct content requests, inline PDF/image previews, attachment downloads, and distinct Preview/Download activity.

### Tests for User Story 2

- [x] T020 [P] [US2] Add authorized list/search tests in `ContosoDashboard-SSD/ContosoDashboard.Tests/Integration/DocumentQueryAuthorizationTests.cs` for title, description, tag, uploader, project, title/date/category/size sorting, category/project/date-range filters, and unauthorized-result exclusion.
- [x] T021 [P] [US2] Add content route tests in `ContosoDashboard-SSD/ContosoDashboard.Tests/Integration/DocumentContentRouteTests.cs` for authenticated access, unauthorized/missing 404 behavior, login redirect, `nosniff`, PDF/JPEG/PNG inline responses, other formats as attachments, and distinct preview/download audit actions.

### Implementation for User Story 2

- [x] T022 [US2] Implement permission-filtered list, sorting, filtering, and search queries in `ContosoDashboard-SSD/ContosoDashboard/Services/DocumentService.cs`; authorize before returning any metadata and search title, description, tags, uploader, and project.
- [x] T023 [US2] Add sorting controls and category/project/date-range filters to `ContosoDashboard-SSD/ContosoDashboard/Pages/Documents.razor`, preserving the authorized query boundary in `DocumentService`.
- [x] T024 [US2] Map `GET /documents/{documentId}/content?download={true|false}` in `ContosoDashboard-SSD/ContosoDashboard/Program.cs` according to `specs/001-document-management/contracts/document-content-route.md`; stream only after current authorization and record Preview versus Download separately.
- [x] T025 [US2] Add the authorized preview/download interaction in `ContosoDashboard-SSD/ContosoDashboard/Pages/DocumentPreview.razor`; render only PDF/JPEG/PNG inline and download all other supported formats as attachments.

**Checkpoint**: US1 and US2 work together; unauthorized callers cannot discover content through list, search, or direct requests.

---

## Phase 5: User Story 3 - Maintain Documents (P2)

**Goal**: Owners edit metadata, replace accepted content safely, and permanently delete documents; project managers and administrators receive only their specified management authority.

**Independent Test**: Exercise metadata changes, replacement success/failure, confirmed deletion, and owner/Project Manager/Administrator authorization; on replacement failure verify the existing accepted file remains unchanged.

### Tests for User Story 3

- [x] T026 [P] [US3] Add metadata authorization and validation tests in `ContosoDashboard-SSD/ContosoDashboard.Tests/Integration/DocumentMetadataTests.cs` for owner edits and denial to unauthorized users.
- [x] T027 [P] [US3] Add replacement/deletion tests in `ContosoDashboard-SSD/ContosoDashboard.Tests/Integration/DocumentLifecycleTests.cs` for scan rejection preserving old content, successful replacement audit, confirmation-required delete, Project Manager project scope, Administrator scope, and permanent content removal.

### Implementation for User Story 3

- [x] T028 [US3] Implement owner-only title/description/category/tag updates with validation in `ContosoDashboard-SSD/ContosoDashboard/Services/DocumentService.cs` and edit controls in `ContosoDashboard-SSD/ContosoDashboard/Pages/Documents.razor`; preserve `Title: Required; 1-255 characters`, `Description: Optional; maximum 2,000 characters`, and the exact category list from `data-model.md`.
- [x] T029 [US3] Implement replacement in `ContosoDashboard-SSD/ContosoDashboard/Services/DocumentService.cs` and `ContosoDashboard-SSD/ContosoDashboard/Services/LocalFileStorageService.cs`; stage and scan the new file first, keep the old accepted file until success, compensate on database failure, and audit the replacement.
- [x] T030 [US3] Implement confirmation-required permanent deletion and owner/Project Manager/Administrator checks in `ContosoDashboard-SSD/ContosoDashboard/Services/DocumentService.cs` and `ContosoDashboard-SSD/ContosoDashboard/Pages/Documents.razor`; remove content and searchable metadata while preserving the activity record.

**Checkpoint**: US3 lifecycle actions are independently authorized and failed replacement cannot damage the current document.

---

## Phase 6: User Story 4 - Share Documents (P2)

**Goal**: Owners share read-only access with selected users or current department members, recipients are notified, and revoked or no-longer-eligible recipients lose access.

**Independent Test**: Share with one user and one department; verify notifications and Shared with Me; change department membership or revoke a share and verify access stops immediately.

### Tests for User Story 4

- [x] T031 [P] [US4] Add user/department share authorization and notification tests in `ContosoDashboard-SSD/ContosoDashboard.Tests/Integration/DocumentShareTests.cs`, including current `Department` membership, revocation, owner-only sharing, read-only recipient access, and denial to unrelated users.

### Implementation for User Story 4

- [x] T032 [US4] Create `ContosoDashboard-SSD/ContosoDashboard/Models/DocumentShare.cs` with `DocumentShareId: Integer primary key`, `DocumentId: Required FK to Document`, `SharedByUserId: Required FK to User`, optional `RecipientUserId` FK, optional `RecipientDepartment`, `SharedAtUtc: Required UTC timestamp`, `RevokedAtUtc: Nullable UTC timestamp; null means active`, and the exact invariant `Exactly one recipient target is set: a user or a department`.
- [x] T033 [US4] Configure DocumentShare relationships, target invariant, indexes, and its additive migration in `ContosoDashboard-SSD/ContosoDashboard/Data/ApplicationDbContext.cs` and `ContosoDashboard-SSD/ContosoDashboard/Data/Migrations/`; do not grant edit, replace, delete, or re-sharing rights through a share.
- [x] T034 [US4] Implement owner-authorized share/revoke and current-department membership checks in `ContosoDashboard-SSD/ContosoDashboard/Services/DocumentService.cs`; resolve department recipients from each user's existing `Department` field on every authorization request.
- [x] T035 [US4] Send share notifications through `ContosoDashboard-SSD/ContosoDashboard/Services/NotificationService.cs` and record share/revoke activity through `ContosoDashboard-SSD/ContosoDashboard/Services/DocumentService.cs`; notify each selected user or current department member.
- [x] T036 [US4] Add the Shared with Me page and navigation in `ContosoDashboard-SSD/ContosoDashboard/Pages/SharedDocuments.razor` and `ContosoDashboard-SSD/ContosoDashboard/Shared/NavMenu.razor`; show only active shares the current user can access.

**Checkpoint**: User and department shares are read-only, immediately revocable, and visible only to eligible recipients.

---

## Phase 7: User Story 5 - Use Documents in Projects, Tasks, and Dashboard (P2)

**Goal**: Users view and attach authorized task/project documents, task uploads inherit the task project, and the dashboard shows recent personal documents and count.

**Independent Test**: Link a document to a task and project, verify authorized task/project views, task-upload project inheritance, five newest personal documents, count, and project notifications.

### Tests for User Story 5

- [x] T037 [P] [US5] Add task/project association and authorization tests in `ContosoDashboard-SSD/ContosoDashboard.Tests/Integration/DocumentTaskTests.cs` for authorized viewing, attaching, project inheritance, and unrelated-user denial.
- [x] T038 [P] [US5] Add dashboard and project notification tests in `ContosoDashboard-SSD/ContosoDashboard.Tests/Integration/DocumentDashboardTests.cs` for the five most recent personal documents, personal count, and eligible project-member notification.

### Implementation for User Story 5

- [x] T039 [US5] Create `ContosoDashboard-SSD/ContosoDashboard/Models/DocumentTask.cs` with `DocumentId: Required FK to Document`, `TaskId: Required FK to TaskItem`, `LinkedByUserId: Required FK to User`, `LinkedAtUtc: Required UTC timestamp`, and `(DocumentId, TaskId)` as the composite primary key.
- [x] T040 [US5] Configure DocumentTask relationships and its additive migration in `ContosoDashboard-SSD/ContosoDashboard/Data/ApplicationDbContext.cs` and `ContosoDashboard-SSD/ContosoDashboard/Data/Migrations/`.
- [x] T041 [US5] Implement authorized task-document linking and task-project inheritance in `ContosoDashboard-SSD/ContosoDashboard/Services/DocumentService.cs`; when uploaded from a task, copy that task's `ProjectId` when present.
- [x] T042 [US5] Add related-document viewing, attachment, and task-origin upload controls in `ContosoDashboard-SSD/ContosoDashboard/Pages/Tasks.razor`; use service authorization for every listed or attached document.
- [x] T043 [US5] Add the authorized project-document section and project-origin links in `ContosoDashboard-SSD/ContosoDashboard/Pages/ProjectDetails.razor`.
- [x] T044 [US5] Add the current user's document count and up to five most recently uploaded personal documents in `ContosoDashboard-SSD/ContosoDashboard/Services/DashboardService.cs` and `ContosoDashboard-SSD/ContosoDashboard/Pages/Index.razor`.
- [x] T045 [US5] Notify eligible project members when a project document is added in `ContosoDashboard-SSD/ContosoDashboard/Services/NotificationService.cs`, using current project membership and avoiding duplicate notifications.

**Checkpoint**: Task, project, and dashboard integrations expose only documents authorized for the current user.

---

## Phase 8: User Story 6 - Review Document Activity (P3)

**Goal**: Administrators review document activity and reports; non-administrators cannot access administrative views or report data.

**Independent Test**: Generate upload, preview, download, delete, and sharing events; verify administrators can review activity and summaries, with Preview and Download represented as distinct actions, and non-admin requests are denied.

### Tests for User Story 6

- [x] T046 [P] [US6] Add administrator report and authorization tests in `ContosoDashboard-SSD/ContosoDashboard.Tests/Integration/DocumentActivityReportTests.cs` for event history, file types, active uploaders, access patterns, distinct preview/download counts, and non-administrator denial.

### Implementation for User Story 6

- [x] T047 [US6] Implement administrator-only activity queries and summaries in `ContosoDashboard-SSD/ContosoDashboard/Services/DocumentService.cs` and `ContosoDashboard-SSD/ContosoDashboard/Pages/DocumentActivity.razor`; include upload, preview, download, delete, and share events. Preserve the rule `No retention period is specified. Do not purge records automatically until a retention policy is agreed`.
- [x] T048 [US6] Add the administrative activity/report page navigation in `ContosoDashboard-SSD/ContosoDashboard/Shared/NavMenu.razor` and enforce Administrator authorization at the service and page boundaries.

**Checkpoint**: Activity and reports are available to Administrators only and do not merge preview and download actions.

---

## Phase 9: Polish & Cross-Cutting Validation

**Purpose**: Verify performance, security, documentation, and the full implementation path.

- [x] T049 [P] Add SQLite-backed performance checks for 500-document list and search targets in `ContosoDashboard-SSD/ContosoDashboard.Tests/Integration/DocumentQueryPerformanceTests.cs`; verify list/search under 2 seconds and preview under 3 seconds using the documented training environment.
- [x] T050 Review and test direct-object authorization, private storage paths, untrusted filenames/content types, scanner fail-closed behavior, `nosniff`, and no-public-file access in `ContosoDashboard-SSD/ContosoDashboard.Tests/Integration/DocumentSecurityTests.cs`.
- [x] T051 Update `ContosoDashboard-SSD/README.md` with local storage location/configuration, loopback ClamAV prerequisite, non-production warning, and the non-destructive SQLite migration procedure.
- [x] T052 Run the build, complete test suite, fresh-database migration, populated legacy-database migration, and end-to-end scenarios in `specs/001-document-management/quickstart.md`; fix failures in the owning files and record any unavailable local ClamAV check. Live ClamAV was unavailable at `127.0.0.1:3310`; automated scanner protocol tests passed.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies; creates the test project and reusable fixtures.
- **Foundational (Phase 2)**: Depends on Setup and blocks all stories; stabilizes seed data and safely transitions SQLite from `EnsureCreated` to migrations without deleting user data.
- **US1 (Phase 3)**: Depends on Foundation; delivers the upload-and-list MVP.
- **US2 (Phase 4)**: Depends on US1's document storage and records; adds authorized queries and content delivery.
- **US3 (Phase 5)**: Depends on US1 and US2 for persisted documents and authorized access.
- **US4 (Phase 6)**: Depends on US1 and US2 for documents and access checks.
- **US5 (Phase 7)**: Depends on US1 and US2 for document persistence and authorized retrieval.
- **US6 (Phase 8)**: Depends on the event-producing behavior in US1-US5.
- **Polish (Phase 9)**: Depends on all stories selected for release.

### User Story Dependencies

`Foundation → US1 → US2 → {US3, US4, US5} → US6 → Polish`

US3, US4, and US5 are logically independent after US2, but their shared `DocumentService.cs`, `ApplicationDbContext.cs`, migration directory, and notification service mean they should be integrated sequentially unless work is split across non-overlapping files.

### Parallel Opportunities

- Setup test-project creation is followed by independent fixture work only after the project exists.
- In US1, T010 and T011 are separate test files and can be authored together; after tests exist, T016 storage and T017 scanner adapters can be implemented in parallel.
- In US2, T020 and T021 are separate test files and can be authored together.
- In US3, T026 and T027 are separate test files and can be authored together.
- In US5, T037 and T038 are separate test files and can be authored together.
- Tasks marked `[P]` are the only tasks planned for parallel execution; shared service/context/migration edits remain sequential.

### Parallel Execution Examples

US1:
- T010: `ContosoDashboard-SSD/ContosoDashboard.Tests/Unit/DocumentFileValidationTests.cs`
- T011: `ContosoDashboard-SSD/ContosoDashboard.Tests/Integration/DocumentUploadTests.cs`

US2:
- T020: `ContosoDashboard-SSD/ContosoDashboard.Tests/Integration/DocumentQueryAuthorizationTests.cs`
- T021: `ContosoDashboard-SSD/ContosoDashboard.Tests/Integration/DocumentContentRouteTests.cs`

US3:
- T026: `ContosoDashboard-SSD/ContosoDashboard.Tests/Integration/DocumentMetadataTests.cs`
- T027: `ContosoDashboard-SSD/ContosoDashboard.Tests/Integration/DocumentLifecycleTests.cs`

US4:
- T031: `ContosoDashboard-SSD/ContosoDashboard.Tests/Integration/DocumentShareTests.cs`

US5:
- T037: `ContosoDashboard-SSD/ContosoDashboard.Tests/Integration/DocumentTaskTests.cs`
- T038: `ContosoDashboard-SSD/ContosoDashboard.Tests/Integration/DocumentDashboardTests.cs`

US6:
- T046: `ContosoDashboard-SSD/ContosoDashboard.Tests/Integration/DocumentActivityReportTests.cs`

---

## Implementation Strategy

### MVP First (User Story 1)

1. Complete Setup and Foundational phases, including the safe SQLite baseline/adoption path.
2. Complete US1 tests before upload models and implementation.
3. Implement private streaming storage and fail-closed scanning, then upload UI and My Documents list.
4. Stop and validate US1 independently using its listed test criteria; do not start the remaining stories until the MVP passes.

### Incremental Delivery

1. Add US2 authorized search, preview, and download.
2. Add US3 document lifecycle controls.
3. Add US4 user/department shares and notifications.
4. Add US5 project/task/dashboard integration.
5. Add US6 administrator activity and reports.
6. Complete cross-cutting performance, security, migration, and quickstart validation.

### Independent Test Criteria by Story

- **US1**: Clean supported upload appears with required metadata; each selected file reports its result; validation and scanner failures create no available document.
- **US2**: Lists/search contain only authorized metadata; direct content requests enforce current authorization; preview/download headers and separate audit actions match the content contract.
- **US3**: Authorized edits persist; only authorized owners/managers/admins can replace/delete; failed replacement preserves old bytes; confirmed delete removes content and search visibility.
- **US4**: Active user/department recipients can read and are notified; revocation or department change removes access; shares never grant edit/delete/re-share.
- **US5**: Task/project views are authorization-filtered; task uploads inherit project; dashboard returns up to five recent personal uploads and the correct personal count; eligible project notifications are created.
- **US6**: Administrators can review and report event types, uploaders, and access patterns; previews and downloads remain distinct; non-administrators are denied.

## Completion Summary

- **Total tasks**: 52
- **By user story**: US1 10; US2 6; US3 5; US4 6; US5 9; US6 3
- **Shared tasks**: Setup 2; Foundational 7; Polish 4
- **Suggested MVP**: US1 after Setup and Foundational phases
- **Format validation**: All task lines use `- [ ] T###`, use `[P]` only where parallel work is safe, include `[US#]` in story phases, and name the target repository path.
