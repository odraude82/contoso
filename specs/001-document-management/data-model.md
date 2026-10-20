# Data Model: Document Upload and Management

Metadata resides in the existing SQLite database. File bytes stay in a private local storage root,
not in SQLite and not under `wwwroot`. All keys follow the application's existing integer-key
convention.

## Document

Represents an accepted, malware-clean document.

| Field | Rules |
|---|---|
| DocumentId | Integer primary key |
| Title | Required; 1-255 characters |
| Description | Optional; maximum 2,000 characters |
| Category | Required text; one of Project Documents, Team Resources, Personal Files, Reports, Presentations, Other |
| OriginalFileName | Required display-only value; never used to form a path |
| RelativeStoragePath | Required generated path relative to the configured private storage root |
| ContentType | Required validated media type for PDF, DOCX, XLSX, PPTX, plain text, JPEG, or PNG; maximum 255 characters; legacy DOC, XLS, and PPT are rejected |
| FileSizeBytes | Required positive size; at most the configured 25 MB limit |
| UploadedByUserId | Required FK to User; the uploader and owner |
| UploadedAtUtc | Required UTC timestamp |
| ProjectId | Optional FK to Project |
| MalwareScannedAtUtc | Required timestamp set only after a clean scan |
| ScannerVersion | Optional scanner engine/signature version for diagnostic audit |

Relationships: many documents belong to one uploader; a document may belong to one project; a document
has many tags, shares, task links, and activity records. Deletion cascades to tags, shares, and task
links but not to activity records.

Indexes: uploader + upload time, project + upload time, category + upload time, and title. Search
queries are always permission-filtered before results are returned.

## DocumentTag

Represents one normalized tag on a document.

| Field | Rules |
|---|---|
| DocumentTagId | Integer primary key |
| DocumentId | Required FK to Document |
| Value | Required, trimmed, case-normalized text; maximum 100 characters |

A document may have many tags. Enforce uniqueness on `(DocumentId, Value)` and index `Value` for
search. A tag does not grant access.

## DocumentShare

Represents an active or revoked share from a document owner.

| Field | Rules |
|---|---|
| DocumentShareId | Integer primary key |
| DocumentId | Required FK to Document |
| SharedByUserId | Required FK to User |
| RecipientUserId | Optional FK to User |
| RecipientDepartment | Optional department/team value matching the application's existing team convention |
| SharedAtUtc | Required UTC timestamp |
| RevokedAtUtc | Nullable UTC timestamp; null means active |

Exactly one recipient target is set: a user or a department. Share access grants read-only access and
is evaluated together with owner authority; a share does not grant edit, replace, delete, or
re-sharing rights. Removing department membership or revoking a share removes access on the next
request.

## DocumentTask

Join entity linking documents and tasks; supports a document used by multiple tasks without duplicating
file metadata.

| Field | Rules |
|---|---|
| DocumentId | Required FK to Document |
| TaskId | Required FK to TaskItem |
| LinkedByUserId | Required FK to User |
| LinkedAtUtc | Required UTC timestamp |

Use `(DocumentId, TaskId)` as the composite primary key. When a document is uploaded from a task, its
ProjectId is copied from that task when present.

## DocumentActivity

Append-only audit event for uploads, rejected uploads, downloads, previews, metadata edits, file
replacements, deletions, and share changes.

| Field | Rules |
|---|---|
| DocumentActivityId | Integer primary key |
| DocumentId | Nullable integer snapshot, not a cascading FK; null supports rejected uploads, and the snapshot lets delete events survive document deletion |
| DocumentTitleSnapshot | Optional title snapshot for events after permanent deletion |
| ActorUserId | Required FK to User |
| Action | Required bounded action name |
| Outcome | Required success/rejected/failed value |
| OccurredAtUtc | Required UTC timestamp |
| TargetSnapshot | Optional recipient or team snapshot, without file content or secrets |

No retention period is specified. Do not purge records automatically until a retention policy is agreed.

## Existing Entity Changes

- Add `Documents` navigation to `User` and `Project` only where it simplifies existing query patterns.
- Add `DocumentTasks` navigation to `TaskItem` and `Document`.
- Extend `NotificationType` with document-shared and project-document-added events; continue using the
  existing `Notification` table and `INotificationService`.
- Keep authorization logic in `DocumentService`; UI visibility is not an authorization boundary.

## Upload State and Consistency

A selected file is copied to a generated-name staging file outside `wwwroot`, streamed without a full
memory buffer, and scanned. A non-clean or indeterminate verdict deletes the staging file and creates
no available Document. A clean file is moved to its final generated path before metadata is committed.
If the SQLite write fails, the service removes the final file as compensation and records the failure
where possible. The original filename is never part of `RelativeStoragePath`.
