# Implementation Plan: Document Upload and Management

**Branch**: `001-document-management` (planned; not checked out) | **Date**: 2026-10-20 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/001-document-management/spec.md`

## Summary

Add an authenticated document workflow to the existing Blazor Server application: local uploads,
metadata, search, project/task integration, controlled sharing, preview/download, and audit reports.
Use EF Core migrations for SQLite metadata and a local filesystem storage service for document
content. Uploads are streamed to a staging directory outside `wwwroot`, scanned by a local ClamAV
daemon, and published only after a clean verdict. Authorization remains in the service boundary.

## Technical Context

**Language/Version**: C# 14, .NET 10.0

**Primary Dependencies**: ASP.NET Core 10 Blazor Server, EF Core SQLite 10.0.13, local ClamAV daemon;
xUnit and ASP.NET Core test hosting in a new test project

**Storage**: SQLite for document metadata and audit records; local files under a configured
`AppData/uploads` root outside `wwwroot`

**Testing**: New xUnit test project; SQLite in-memory with an open connection, temporary filesystem
roots, fake scanner, authorization tests, and optional live ClamAV/EICAR integration test

**Target Platform**: Windows training workstation and supported desktop browsers; runtime workflows
must operate without Internet or cloud services

**Project Type**: Existing ASP.NET Core Blazor Server web application with internal authenticated
content endpoint

**Performance Goals**: Upload a 25 MB file within 30 seconds; list up to 500 documents within 2
seconds; search within 2 seconds; preview supported PDFs and images within 3 seconds

**Constraints**: Preserve mock cookie authentication and role checks; enforce a server-side 25 MB
per-file limit; use generated storage names and validated file types; keep files outside public
static content; malware scanning must fail closed; no cloud storage or scanner dependency

**Scale/Scope**: Six user journeys; up to 500 documents for list performance; multi-file uploads,
project/task links, user/department sharing, notifications, and administrative activity reports

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- **Training-only scope**: PASS. The plan does not claim production readiness or replace mock auth.
- **Security and authorization**: PASS. Access checks are centralized; file paths are private;
  validation and scanning precede publication; preview/download are authorized.
- **Offline-first infrastructure**: PASS. SQLite, local files, and a pre-provisioned local scanner
  work without cloud services. Signature updates are an environment setup concern, not a runtime
  service dependency.
- **Layered architecture and simplicity**: PASS. Reuse the existing data, service, and page layers;
  add one document service, storage/scanner adapters, and one internal content route. No Azure SDK,
  separate API project, or YARA layer is planned.
- **Verifiable changes**: PASS. Add focused unit and integration tests because the repository has no
  existing test project.

**Gate result**: PASS, conditional on the local scanner being configured and on an explicit,
backed-up transition for databases previously created with `EnsureCreated`.

## Project Structure

### Documentation (this feature)

```text
specs/001-document-management/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   └── document-content-route.md
└── tasks.md
```

### Source Code (repository root)

```text
ContosoDashboard-SSD/
├── ContosoDashboard/
│   ├── Data/
│   │   ├── ApplicationDbContext.cs
│   │   └── Migrations/
│   ├── Models/
│   │   ├── Document.cs
│   │   ├── DocumentActivity.cs
│   │   ├── DocumentShare.cs
│   │   ├── DocumentTag.cs
│   │   └── DocumentTask.cs
│   ├── Services/
│   │   ├── DocumentService.cs
│   │   ├── LocalFileStorageService.cs
│   │   └── ClamAvMalwareScanner.cs
│   ├── Pages/
│   │   ├── Documents.razor
│   │   ├── Index.razor
│   │   ├── ProjectDetails.razor
│   │   └── Tasks.razor
│   ├── Shared/
│   │   └── NavMenu.razor
│   └── Program.cs
└── ContosoDashboard.Tests/
    ├── Unit/
    └── Integration/
```

**Structure Decision**: Extend the single existing web project. Keep metadata access and permission
decisions in `DocumentService`; keep disk operations and malware scanning behind separate service
interfaces. Add the download/preview route to the existing host rather than creating a public API
project. Add one test project for service, storage, authorization, and end-to-end coverage.

## Complexity Tracking

No constitution violations or unjustified architectural layers are planned. The test project is
required to verify security and storage behavior and is not a production application project.
