# Research: Document Upload and Management

## Local Malware Scanning

**Decision**: Use a scanner adapter that talks to a locally installed ClamAV daemon. Bind its TCP
listener to loopback only. The application stages each upload outside `wwwroot`, scans the staged
file, and publishes it only after a clean verdict. Infected, timed-out, unavailable, or indeterminate
scans fail closed: remove the staged file, report a generic upload failure, and record the outcome
without exposing scanner details to the user. Keep the scanner behind `IMalwareScanner` so tests use
controlled clean/infected/error verdicts.

**Rationale**: ClamAV documents `clamd` as a multi-threaded scanning daemon and `clamdscan` as its
client. Its official documentation warns that TCP socket traffic is unauthenticated, so the service
must remain bound to loopback. Pre-provision signatures locally; upload workflows must not call a
cloud scanner or require an Internet connection. An application adapter makes scanner timeouts and
error handling deterministic and keeps the UI independent of the scanner process.

**Alternatives considered**:
- `clamscan` per file: simpler process model, but initializes the scanning engine and signatures
  for each invocation; not preferred for repeated uploads.
- YARA in addition to ClamAV: adds rule maintenance and a second scanning path without a requirement
  that justifies the extra complexity.
- Cloud scanning services: rejected because the training feature must work offline.
- Allowing uploads when scanning fails: rejected because it violates FR-008 and the security gate.

## Blazor Server Upload Handling

**Decision**: Read `IBrowserFile` with an explicit maximum size and stream it directly to a uniquely
named staging file. Process the selected files sequentially, show per-file progress and outcome, and
never buffer complete files in a `MemoryStream`. Validate the file size on the server, allow-list
extensions and file signatures, and treat client name and MIME type as untrusted metadata. Use a
bounded multiple-file selection (initial default: 10 files per batch) to limit resource consumption.

**Rationale**: Microsoft Learn recommends streaming server-side uploads to a `FileStream`, passing an
explicit maximum to `OpenReadStream`, and not trusting client-supplied names or sizes. In Blazor
Server, file bytes travel over SignalR as bounded stream messages; the initial file-selection
metadata is a separate message. Keep SignalR's normal message limit rather than raising it to 25 MB;
the bounded batch size limits metadata, and a focused test should confirm that the initial selection
fits the configured limit. Only adjust `HubOptions.MaximumReceiveMessageSize` if that metadata test
demonstrates a need. Kestrel request-body and multipart limits apply if a separate HTTP multipart
upload endpoint is introduced, not to the current direct Blazor Server stream.

**Alternatives considered**:
- Copying each full upload into memory: rejected because 25 MB files and multiple selections create
  avoidable memory pressure and denial-of-service risk.
- Saving directly to the final path before scanning: rejected because an unverified file must not be
  visible or downloadable.
- Trusting browser-reported size or MIME type: rejected because those values are supplied by the
  client.

## Filesystem and SQLite Consistency

**Decision**: Treat file storage and SQLite as separate resources. Upload flow: write a generated-name
file to a private staging area, scan it, move it to its final generated path, then save document
metadata and audit activity in a database transaction. If the database write fails, remove the final
file as compensation. For delete, authorize and confirm first, record the activity, remove content,
and ensure no deleted metadata remains discoverable; log cleanup failures for operator action.

**Rationale**: A database transaction cannot atomically include filesystem operations. Explicit staging
and compensation prevent visible metadata for missing files and reduce untracked files. Relative paths
remain portable and original filenames are display metadata only.

**Alternatives considered**:
- Store file bytes in SQLite: rejected because the requirements call for local filesystem storage and
  files are as large as 25 MB.
- Make the upload directory public: rejected because every preview and download must enforce
  authorization.

## SQLite Schema Evolution

**Decision**: Make EF Core migrations the canonical schema mechanism for the feature and future changes;
do not combine `EnsureCreated` and migrations. Stabilize the existing seed data before generating the
baseline because the current `HasData` values use `DateTime.UtcNow`. Create a baseline migration for
new databases. For an existing `EnsureCreated` database, require a backup, verify that its schema
matches the baseline, create/record the baseline migration history entry without replaying the
already-created tables, then apply the additive document migration. Never drop an existing database
automatically.

**Rationale**: Microsoft Learn states that `EnsureCreated` initializes only when no tables exist,
that it does not work well with migrations, and that transition is not seamless. EF's documented
squashing procedure requires a backup and manually recording an already-present schema as the new
initial migration. Test both a fresh database and a populated legacy database before rollout.

**Alternatives considered**:
- Keep `EnsureCreated`: rejected because it will not add the new tables to an already-created SQLite
  file.
- Drop and recreate automatically: rejected because it destroys local user data without consent.
- Run the baseline migration against an existing schema: rejected because it attempts to create
  tables that already exist.

## Query and Test Strategy

**Decision**: Begin with indexed relational queries in SQLite for the expected 500-document list and
search scope; add full-text search only if measured performance misses the two-second requirement. Add
an xUnit test project using an open in-memory SQLite connection, temporary file roots, and a fake
scanner. Keep a separate optional integration test for a configured local ClamAV service using a
standard harmless antivirus test signature.

**Rationale**: The data volume in the spec is modest, so a full-text engine is not initially justified.
The existing repository has no test project. SQLite-backed tests exercise provider behavior better
than an in-memory EF provider, while temporary paths and scanner fakes make failure modes repeatable.

**Alternatives considered**:
- Add a search engine or SQLite FTS immediately: rejected until performance tests demonstrate a need.
- Use only mocked persistence tests: rejected because relationship, migration, and query behavior must
  be verified against SQLite.

## References

- [ASP.NET Core Blazor file uploads (.NET 10)](https://learn.microsoft.com/en-us/aspnet/core/blazor/file-uploads?view=aspnetcore-10.0)
- [EF Core Create and Drop APIs](https://learn.microsoft.com/en-us/ef/core/managing-schemas/ensure-created)
- [EF Core Managing Migrations](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/managing)
- [ClamAV Scanning](https://docs.clamav.net/manual/Usage/Scanning.html)
- [ClamAV Configuration](https://docs.clamav.net/manual/Usage/Configuration.html)
