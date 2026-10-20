<!--
Sync Impact Report
Version change: 1.0.0 -> 1.0.1 (PATCH: technical baseline updated)
Modified principles: None
Modified sections: Technical and Operational Constraints (ASP.NET Core 8 / SQL Server -> ASP.NET Core 10 / SQLite)
Added sections: None
Removed sections: None
Follow-up TODOs: Confirm the original ratification date; remove this report before commit.
-->
# ContosoDashboard Constitution

## Core Principles

### I. Training-Only Scope
ContosoDashboard and its architecture MUST be treated as educational examples, not as production-ready software. Changes MUST preserve this limitation in user-facing and developer documentation when relevant. The mock authentication, local storage, and other training implementations MUST NOT be represented as suitable production controls.

### II. Security and Authorization
Protected pages and data operations MUST enforce authentication and authorization, including role and resource-membership checks at the service boundary. Implementations MUST validate untrusted input, prevent unauthorized direct-object access, and keep uploaded files outside web-accessible paths. User-provided file names MUST NOT be used as storage paths. These controls improve the training example but do not make its mock identity system production-ready.

### III. Offline-First and Replaceable Infrastructure
Core application workflows MUST run locally without requiring cloud services or external service dependencies. Infrastructure dependencies that may change between training and future implementations MUST be accessed through interfaces, keeping business logic independent of the implementation. A cloud migration path MUST NOT be described as implemented unless it is actually available and verified.

### IV. Layered Architecture and Simplicity
UI pages, business services, and data persistence MUST retain distinct responsibilities and use the existing application layers. Changes MUST prefer the smallest solution consistent with the feature requirements and MUST avoid major rewrites without an approved need. Shared behavior belongs in the appropriate service or data abstraction rather than duplicated page logic.

### V. Verifiable Changes
Behavior changes and bug fixes MUST include automated tests for important behavior when a suitable test boundary exists, especially authorization and data-access rules. When automated coverage is impractical, the change MUST document reproducible verification steps. Before completion, contributors MUST run the relevant build and available checks and MUST report any checks that could not be run.

## Technical and Operational Constraints

The application targets ASP.NET Core 10.0 with Blazor Server, Entity Framework Core 10, and SQLite. Changes MUST remain compatible with this baseline unless a separately approved modernization changes it. Training workflows MUST work offline with local development dependencies; SQLite stores the development database locally, and no cloud dependency is mandatory. Documentation MUST distinguish current capabilities from future migration examples and disclose known training limitations.

## Development Workflow

Feature work MUST follow the repository's Spec-Driven Development workflow: clarify requirements, record the specification and implementation plan, break work into verifiable tasks, and assess the result against the agreed artifacts. Reviews MUST check applicable principles, authorization boundaries, and test evidence. Documentation and stakeholder requirements MUST be updated when implementation changes their stated behavior or constraints.

## Governance

This constitution governs project development practices. Amendments MUST be proposed as changes to this document, reviewed and approved by the project maintainer, and accompanied by an updated Sync Impact Report during review. Every change MUST be checked for compliance with the principles during specification and review; exceptions MUST be explicitly documented with their rationale and scope.

The constitution uses semantic versioning. A MAJOR increment records a backward-incompatible governance change or removal/redefinition of a principle; a MINOR increment records a new principle or materially expanded guidance; a PATCH increment records a clarification or non-semantic refinement. Compliance review is required for specifications, code reviews, and completion checks affected by a change.

**Version**: 1.0.1 | **Ratified**: TODO(RATIFICATION_DATE): original adoption date unknown | **Last Amended**: 2026-10-20
