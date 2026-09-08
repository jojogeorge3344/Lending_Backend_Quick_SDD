# Feature Specification: Loan Submission API

**Feature Branch**: `001-loan-submission-api`

**Created**: 2026-09-08

**Status**: Draft

**Input**: User description: "Create a REST API for submitting a loan.

The API will need the following data.
- Customer data: full name, ID number, address (street, city, zip code), birth date, phone number, email, and current monthly income.
- Collateral data is always a car: brand, model, manufacturing year, and license plate.
- Proposed loan data: amount (must be a multiple of 100), tenure (between 3-48 months).

If all inputs are valid, store the data in the database as \"IN_PROCESS\" status."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Submit a complete, valid loan application (Priority: P1)

A loan applicant provides their personal details, their car's collateral details, and the loan
terms they want (amount and tenure) in a single submission, so the lender can begin processing
their request.

**Why this priority**: This is the core capability the feature exists to deliver — without it,
there is no way to start a loan application at all.

**Independent Test**: Submit a payload containing fully valid customer, collateral, and loan
data. Verify a new loan application record is created with status "IN_PROCESS" and a reference
identifier is returned to the caller.

**Acceptance Scenarios**:

1. **Given** valid customer data, valid car collateral data, a loan amount that is a positive
   multiple of 100, and a tenure between 3 and 48 months, **When** the application is submitted,
   **Then** the system stores a new loan application with status "IN_PROCESS" and returns a
   reference identifier for it.
2. **Given** a successfully submitted application, **When** the applicant or system looks it up
   by its returned reference identifier immediately afterward, **Then** the stored customer,
   collateral, and loan data match exactly what was submitted.

---

### User Story 2 - Reject incomplete or invalid submissions with clear feedback (Priority: P2)

A loan applicant submits an application that is missing required information or contains
invalid values (e.g., a malformed email address), and is told specifically what is wrong so they
can correct and resubmit.

**Why this priority**: Applicants and integrating clients need actionable feedback; without it,
failed submissions are a dead end and generate support burden.

**Independent Test**: Submit payloads that are each missing one required field, or that contain
one invalid field value, and confirm each is rejected with an error identifying the offending
field, while no loan application record is created.

**Acceptance Scenarios**:

1. **Given** a submission missing the customer's email address, **When** it is submitted,
   **Then** the system rejects it and indicates the email field is required, and no loan
   application record is created.
2. **Given** a submission with a malformed email address or phone number, **When** it is
   submitted, **Then** the system rejects it and identifies which field is invalid.
3. **Given** a submission missing any collateral field (brand, model, manufacturing year, or
   license plate), **When** it is submitted, **Then** the system rejects it and identifies the
   missing field, and no loan application record is created.

---

### User Story 3 - Enforce loan amount and tenure business rules (Priority: P3)

A loan applicant's proposed amount and tenure are checked against the lender's business rules
(amount must be a multiple of 100; tenure must be between 3 and 48 months) before any
application is accepted, so terms outside acceptable bounds are never processed further.

**Why this priority**: These are explicit, financially significant business rules; enforcing them
consistently — including at their boundaries — prevents invalid loan terms from ever reaching
downstream processing.

**Independent Test**: Submit otherwise-valid applications with amount/tenure values at and
outside the allowed boundaries, and confirm acceptance/rejection matches the rule exactly.

**Acceptance Scenarios**:

1. **Given** an amount that is not a whole multiple of 100 (e.g., 1050), **When** submitted,
   **Then** the system rejects it and indicates the amount must be a multiple of 100.
2. **Given** a tenure of 2 months or 49 months, **When** submitted, **Then** the system rejects
   it and indicates tenure must be between 3 and 48 months.
3. **Given** a tenure of exactly 3 months or exactly 48 months (the inclusive boundaries), with
   all other data valid, **When** submitted, **Then** the system accepts the application.

---

### Edge Cases

- What happens when the loan amount is zero, negative, or a non-numeric value? Rejected as
  invalid; no application record is created.
- What happens when the customer's birth date indicates they are under 18 at submission time?
  Rejected as invalid.
- What happens when the collateral car's manufacturing year is in the future, or is implausibly
  old? Rejected as invalid.
- What happens when the same customer (same ID number) submits more than one application?
  Each submission is validated and processed independently; no deduplication is performed (see
  Assumptions).
- What happens when a request is malformed (not valid structured data) or omits an entire
  section (e.g., no collateral data at all)? Rejected as invalid, equivalent to missing required
  fields.
- What happens if a submission fails on multiple fields at once? The system reports all
  identifiable field-level failures, not just the first one encountered.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST provide an API operation that accepts, in a single submission, customer
  data, car collateral data, and proposed loan data for a new loan application.
- **FR-002**: System MUST require and validate the following customer fields as mandatory: full
  name, ID number, address (street, city, zip code), birth date, phone number, email address, and
  current monthly income.
- **FR-003**: System MUST require and validate the following collateral fields as mandatory,
  given that collateral is always a car: brand, model, manufacturing year, and license plate.
- **FR-004**: System MUST require a proposed loan amount and reject any submission where the
  amount is not a positive whole-number multiple of 100.
- **FR-005**: System MUST require a proposed loan tenure and reject any submission where the
  tenure (in months) is outside the inclusive range of 3 to 48.
- **FR-006**: System MUST validate that the email address and phone number are in a valid,
  well-formed format before accepting a submission.
- **FR-007**: System MUST validate that the customer's birth date indicates an age of at least 18
  years at the time of submission.
- **FR-008**: System MUST validate that the collateral car's manufacturing year is a plausible
  year (not in the future, and not before 1900).
- **FR-009**: System MUST validate current monthly income as a non-negative numeric value.
- **FR-010**: System MUST reject the entire submission — persisting nothing — if any one or more
  required fields are missing or fail validation, and MUST identify every failing field in the
  rejection response.
- **FR-011**: System MUST NOT persist any customer, collateral, or loan data for a submission
  that fails validation.
- **FR-012**: Upon successful validation of all fields, System MUST create and persist a new loan
  application record with status "IN_PROCESS".
- **FR-013**: Upon successful creation, System MUST persist the submitted customer and collateral
  data linked to the created loan application.
- **FR-014**: Upon successful creation, System MUST return a unique reference identifier for the
  new loan application to the caller.
- **FR-015**: System MUST reject a proposed loan amount below 100; there is no upper bound on the
  loan amount beyond the multiple-of-100 rule in FR-004.

### Key Entities *(include if feature involves data)*

- **Loan Application**: The record created by a submission. Key attributes: reference identifier,
  status (starts as "IN_PROCESS"), proposed loan amount, proposed tenure (months). Related to
  exactly one Customer and one Collateral (Car).
- **Customer**: The applicant's personal and financial profile as of submission. Key attributes:
  full name, ID number, address (street, city, zip code), birth date, phone number, email
  address, current monthly income.
- **Collateral (Car)**: The vehicle offered as collateral for the loan. Key attributes: brand,
  model, manufacturing year, license plate.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: An applicant submitting a complete, valid application receives confirmation,
  including a reference identifier, within 5 seconds.
- **SC-002**: 100% of submissions containing at least one missing or invalid required field are
  rejected without creating any stored loan application, customer, or collateral record.
- **SC-003**: 100% of successfully validated submissions result in a loan application stored with
  status "IN_PROCESS" and immediately retrievable by its reference identifier.
- **SC-004**: 100% of rejected submissions include, in the response, an explicit identification
  of every field that failed validation.
- **SC-005**: Loan amount and tenure boundary rules (multiple of 100; 3-48 months inclusive) are
  enforced with zero exceptions across all submissions, including at exact boundary values.

## Assumptions

- The applicant must be at least 18 years old at the time of submission; this is the standard
  legal age of majority for entering a financial contract and no alternate age was specified.
- The collateral car's manufacturing year must fall between 1900 and the current year; this
  bounds an otherwise-unconstrained numeric field to plausible real-world values.
- No deduplication or uniqueness enforcement is performed on customer ID number: the same
  customer may have multiple independent loan applications, each validated and stored on its own
  merits.
- This feature covers only the submission, validation, and initial storage of a loan application
  in "IN_PROCESS" status. The subsequent review/underwriting workflow that moves an application
  out of "IN_PROCESS" (approval, rejection, etc.) is out of scope for this feature.
- Submission requires the caller to be an authenticated client of the platform, consistent with
  the platform's existing authentication policy; this feature does not introduce new
  authentication mechanics.
- Loan amounts are handled in a single, system-wide currency; multi-currency support is out of
  scope for this feature.
- Customer ID number is treated as a required opaque identifier string; no country-specific
  format validation is enforced beyond it being present and non-empty.
- The minimum loan amount is 100 (the smallest positive multiple of 100); there is no maximum
  loan amount enforced by this feature.
