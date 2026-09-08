<!--
Sync Impact Report
==================
Version change: 1.1.0 → 1.2.0
Rationale: MINOR bump — materially expanded Principle I (API Contract Discipline)
with a new, non-negotiable timestamp-format rule for all timestamp input/output
(RFC 3339 / ISO 8601, second precision, explicit numeric UTC offset), plus a
matching Technology Stack Requirements bullet. No principle was removed or
redefined.

Modified principles:
- I. API Contract Discipline — added mandatory timestamp format requirement
  (e.g. `2026-05-05T14:30:00+07:00`) for all request/response, log, and
  audit-column timestamp values.

Added sections: n/a (existing sections expanded, none newly added)

Removed sections: n/a

Templates requiring updates:
- .specify/templates/plan-template.md — ⚠ pending manual check for alignment
  with Constitution Check gates (not modified by this command; verify at next
  /speckit-plan run)
- .specify/templates/spec-template.md — ✅ no constitution-specific references
- .specify/templates/tasks-template.md — ✅ no constitution-specific references

Deferred / TODO placeholders:
- RATIFICATION_DATE remains 2026-09-08 (unchanged from initial adoption; no
  earlier date exists in repo history).
-->

# Lending Backend Constitution
<!-- Go REST API + PostgreSQL service constitution -->

## Core Principles

### I. API Contract Discipline
Every HTTP endpoint MUST be defined by an explicit, versioned contract (OpenAPI/Swagger
or an equivalent schema) before or alongside implementation, and that contract is the
source of truth reviewed in PRs — handlers MUST NOT diverge from it silently. Routes
MUST follow RESTful conventions (resource-oriented paths, correct verbs and status
codes) and MUST be versioned under `/api/v{n}/...`; breaking changes to a published
version are prohibited — ship a new version instead. All request/response bodies MUST
be validated against typed Go structs with explicit validation tags; unvalidated
`interface{}`/`map[string]any` payloads are not permitted on external boundaries.
Every timestamp value crossing an input or output boundary — API request/response
fields, logs, and database audit columns — MUST be formatted as RFC 3339/ISO 8601
with second precision and an explicit numeric UTC offset, e.g.
`2026-05-05T14:30:00+07:00`; bare dates, Unix epoch numbers, or offset-less
("floating") timestamps are not permitted on any boundary.
Rationale: a lending backend has external consumers (web/mobile clients, partner
integrations) who depend on stable, predictable contracts; silent drift causes
production incidents and erodes trust with financial-data consumers.

### II. Test-First Development (NON-NEGOTIABLE)
Tests MUST be written before implementation for every new handler, service method,
and repository function: write the test, confirm it fails, then implement until it
passes (Red-Green-Refactor). Table-driven tests are the default style for Go unit
tests. Every exported package function that touches business logic (loan calculations,
interest, eligibility, repayment schedules) MUST have unit tests covering boundary
conditions (zero, negative, max values) in addition to the happy path. PRs that add
behavior without accompanying tests MUST be rejected in review.
Rationale: lending calculations are financially consequential and hard to verify by
inspection alone; tests are the enforceable proof that logic is correct and stays
correct as the code evolves.

### III. Database Integrity & Migration Safety
All schema changes MUST go through versioned, forward-only migration files checked
into the repository (no manual/ad-hoc DDL against any environment). Every migration
MUST have a corresponding rollback or be explicitly documented as irreversible with
justification. Multi-statement writes that must succeed or fail together (e.g.
creating a loan and its first repayment schedule entry) MUST run inside a single
database transaction. Foreign key constraints, NOT NULL, and CHECK constraints MUST
be used at the database level to enforce invariants — application-level validation
alone is not sufficient for financial data. Money/amount columns MUST use a
fixed-point type (e.g. `NUMERIC`/`DECIMAL`), never floating-point. Every table MUST
include the standard audit columns — `created_at`, `created_by`, `updated_at`,
`updated_by`, `deleted` (boolean), and `deleted_at` — created by the same migration
that creates the table; no table is exempt. Deletes on audited tables MUST be soft
deletes (`deleted = true`, `deleted_at` set) via application/service code, not `DELETE`
statements, and queries MUST filter out soft-deleted rows by default.
Rationale: PostgreSQL is the system of record for money movement; integrity bugs at
this layer cause silent, hard-to-reconcile financial errors, so the database itself
must enforce correctness, not just the application layer.

### IV. Security & Data Protection
All endpoints except explicitly documented public/health routes MUST require
authentication, and authorization checks MUST be enforced at the service layer, not
only inferred from routing. Secrets (DB credentials, signing keys, third-party API
keys) MUST be loaded from environment/secret-manager configuration and MUST NOT be
committed to the repository. All SQL access MUST use parameterized queries or a
query builder/ORM that parameterizes automatically — string-concatenated SQL is
prohibited. Personally identifiable information and financial identifiers (SSNs,
account numbers, full card numbers) MUST be encrypted at rest and MUST NOT appear in
logs; logs and error messages MUST redact sensitive fields.
Rationale: this system handles regulated financial and personal data; a security
lapse here has legal, financial, and customer-trust consequences beyond typical
software defects.

### V. Observability & Structured Logging
Every service MUST emit structured (JSON) logs with a correlation/request ID
propagated across a single request's handler → service → repository call chain.
Every external-facing and inter-service call MUST have timeout and error-path
logging; panics MUST be recovered at the HTTP middleware layer and converted into a
logged 5xx response, never a crashed process. Health and readiness endpoints MUST be
exposed for every deployable service.
Rationale: production issues in a financial backend must be diagnosable after the
fact without reproducing them live; structured, correlated logs are the minimum bar
for that.

### VI. Simplicity & Dependency Discipline
Code MUST follow idiomatic Go (`gofmt`/`go vet`/lint-clean) and standard project
layout conventions. New third-party dependencies MUST be justified in the PR
description over hand-rolled or standard-library alternatives; dependencies that
duplicate functionality already in use MUST NOT be introduced. Abstractions
(interfaces, generic layers, plugin systems) MUST NOT be added ahead of an actual
second implementation or concrete requirement — YAGNI applies. Package boundaries
MUST reflect domain concepts (e.g. `loan`, `repayment`, `ledger`), not technical
layering alone.
Rationale: unnecessary complexity slows down the audits and reviews that a financial
codebase requires more often than typical software, and Go's idioms exist precisely
to keep services like this maintainable by any engineer who picks them up.

## Technology Stack Requirements

- Language: Go (a single supported minor version is pinned in `go.mod`; upgrades are
  deliberate, reviewed changes, not incidental).
- Database: PostgreSQL is the only relational datastore for transactional/financial
  data; migrations are managed by a single chosen tool (e.g. `golang-migrate`,
  `goose`) applied consistently across all environments.
- HTTP layer: a single chosen router/framework (e.g. standard `net/http` with a
  minimal router, `chi`, or `gin`) is used consistently; mixing routing frameworks
  within the same service is prohibited.
- Configuration: environment-variable or secret-manager based configuration only; no
  environment-specific code branches (`if env == "prod"`) outside a dedicated config
  loader.
- All monetary amounts are represented as integer minor units (e.g. cents) or
  `NUMERIC`/`decimal.Decimal` in Go — never `float32`/`float64`.
- Every table's creation migration MUST include the audit columns `created_at`,
  `created_by`, `updated_at`, `updated_by`, `deleted` (boolean, default `false`), and
  `deleted_at` (nullable), per Principle III.
- All timestamp columns and all timestamp fields in API payloads/logs are stored and
  serialized as RFC 3339/ISO 8601 with second precision and an explicit numeric UTC
  offset (e.g. `2026-05-05T14:30:00+07:00`), per Principle I; use `timestamptz` in
  PostgreSQL and `time.Time` with RFC3339 marshaling in Go.

## Development Workflow & Quality Gates

- Every PR MUST pass: `go build`, `go vet`, the configured linter, and the full test
  suite in CI before merge; failing CI blocks merge with no exceptions.
- Every PR touching a handler, service method, or migration MUST include or update
  tests per Principle II; reviewers MUST verify test coverage of the change, not just
  presence of some test.
- Every PR that adds or changes an endpoint MUST include the corresponding API
  contract update (Principle I) in the same PR.
- Every PR that adds or changes a migration MUST include the rollback path or an
  explicit, reviewed justification for why none exists (Principle III).
- At least one other engineer MUST review and approve a PR before merge; the author
  MUST NOT merge their own PR.

## Governance

This constitution supersedes ad-hoc conventions and prior undocumented practice for
this repository. Any conflict between this document and other project documentation
is resolved in favor of this constitution until that other document is updated to
match.

**Amendment procedure**: propose the change via a PR modifying this file, including
a filled-out Sync Impact Report (as an HTML comment at the top of the file) covering
version bump rationale, modified/added/removed sections, and any follow-up impact on
templates or workflow docs. The PR requires the same review approval as any other
change (see Development Workflow above) before merge.

**Versioning policy**: semantic versioning applies to this document.
- MAJOR: backward-incompatible governance changes, or removal/redefinition of an
  existing principle.
- MINOR: a new principle or section is added, or existing guidance is materially
  expanded.
- PATCH: clarifications, wording, typo fixes, and non-semantic refinements.

**Compliance review**: every PR is expected to be reviewable against these
principles; a reviewer who finds a violation MUST block the PR until it is resolved
or an explicit, documented exception is agreed with the team. There is no separate
scheduled audit — compliance is enforced continuously through PR review.

**Version**: 1.2.0 | **Ratified**: 2026-09-08 | **Last Amended**: 2026-09-08
