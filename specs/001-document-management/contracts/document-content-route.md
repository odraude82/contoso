# Internal Content Route Contract

This route is consumed by the ContosoDashboard browser UI. It is not a public integration API.

## Request

`GET /documents/{documentId}/content?download={true|false}`

- Authentication: existing cookie authentication is required.
- `documentId`: positive integer document key.
- `download`: optional boolean; defaults to `false` (preview intent).
- The route delegates access checks to `IDocumentService`; it does not accept filesystem paths or
  user-supplied filenames.
- Each successfully authorized content response records an activity event: `Preview` when
  `download=false` and `Download` when `download=true`. These actions remain distinct in reports.

## Authorization

A caller may receive content only if the current service authorization rules grant read access as the
document owner, a project member, a Team Lead in the uploader's department, an active share recipient,
a Project Manager with project authority, or an Administrator. Edit/delete roles do not imply direct
file-path access.

## Responses

- `200 OK`: streams the document content. `Content-Type` comes from validated document metadata;
  `X-Content-Type-Options: nosniff` is set. Inline display is permitted only for PDF, JPEG, and PNG.
  Other supported file types are always returned as attachments.
- Authenticated but missing or unauthorized document: `404 Not Found`, without revealing whether a
  document exists or disclosing its path.
- Unauthenticated request: follow the existing cookie middleware behavior and redirect to `/login`.
- Missing content on disk: return a generic not-found response, log the incident, and do not disclose
  the storage path.

The endpoint never serves the storage root through static-file middleware. Range requests and public
or anonymous links are out of scope.
