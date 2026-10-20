# Quickstart: Document Management Validation

## Prerequisites

- .NET 10 SDK.
- The ContosoDashboard project and SQLite configuration.
- A local ClamAV daemon with a signature database, bound only to loopback and configured through the
  application's scanner settings. The app rejects uploads when the scanner is unavailable or cannot
  return a clean verdict.
- The feature test project created during implementation.

## Build and Automated Tests

From the workspace root:

```powershell
dotnet build .\ContosoDashboard-SSD\ContosoDashboard\ContosoDashboard.csproj
dotnet test .\ContosoDashboard-SSD\ContosoDashboard.Tests\ContosoDashboard.Tests.csproj
```

Expected result: the application builds, and unit/integration tests pass using an isolated SQLite
in-memory database, temporary upload directories, and a fake malware scanner. Live ClamAV checks are
separately marked and require the local daemon.

## Run the Application

```powershell
dotnet run --project .\ContosoDashboard-SSD\ContosoDashboard\ContosoDashboard.csproj --urls http://localhost:5080
```

Open `http://localhost:5080` and sign in using one of the seeded training users.

## End-to-End Scenarios

1. **Accepted upload**: Sign in as an Employee; upload a supported PDF within the 25 MB limit with a
   title and category; verify progress, success, captured metadata, and visibility in My Documents.
2. **Validation failures**: Try a file over the limit, a disallowed extension, a mismatched file
   signature, and a harmless antivirus test signature. Verify each file is rejected and cannot be
   previewed, downloaded, searched, or associated with a task.
3. **Scanner fail-closed**: Stop or misconfigure the local scanner; upload a supported file; verify the
   UI reports failure and no accepted metadata or public file is created.
4. **Authorization matrix**: Verify owner, project member, same-department Team Lead, Project Manager,
   Administrator, active share recipient, and unrelated user access. Confirm Team Leads have only
   read/download access to team members' documents unless another role independently authorizes an
   action.
5. **Search and organization**: Create at least 500 test documents; verify sorting, filtering, tags,
   and authorized search results meet their two-second target.
6. **Project/task/dashboard integration**: Upload from a task and verify project association; verify
   project document access, recent five personal documents, count, and project notifications.
7. **Delete and audit**: Confirm deletion removes the file and document from lists/search while the
   audit event remains available to Administrators.
8. **Preview and download**: Verify PDF/JPEG/PNG render inline; other formats download as attachments;
   unauthorized requests return no content and reveal no storage path.

## SQLite Upgrade Validation

Back up an existing local `ContosoDashboard.db` before the feature upgrade. Validate two paths:

- Fresh database: apply the baseline and feature migrations; verify existing seed data and document
  tables are created.
- Existing `EnsureCreated` database: verify its schema against the baseline, record the baseline as
  already applied using the documented one-time upgrade procedure, then apply only the additive
  document migration. Confirm existing users, projects, and tasks remain intact.

Never delete or recreate a user's SQLite file automatically as part of startup.
