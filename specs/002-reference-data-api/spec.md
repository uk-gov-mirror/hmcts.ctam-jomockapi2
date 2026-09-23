# Feature Specification: Reference Data API — Appointment Titles

**Feature Branch**: `002-reference-data-api`

**Created**: 2026-09-23

**Status**: Draft

**Input**: User description: "Create a feature specification for the Reference Data API. Implement the first usable slice of the generic Reference Data capability (Constitution Principle VII), supporting only `appointment_titles` → AppointmentTitle, plus the deprecated alias `appointment_title`, exposed under `/api/v1/reference_data`. Provide collection retrieval (`GET /api/v1/reference_data/{attribute}`) and single-record retrieval (`GET /api/v1/reference_data/{attribute}/{reference_id}`), with canonical and alias routes behaving equivalently. Use the supplied E-Links Swagger as the behavioural reference; synthetic deterministic data; centralised configurable authentication returning 401 on missing/invalid credentials; shared validation and error contract; extensibility for the remaining ten reference-data types without implementing them; mark gaps in the E-Links contract as clarifications rather than inventing behaviour."

## Clarifications

### Session 2026-09-23

- Q: Should AppointmentTitle records include a `name` field, which the Swagger `ReferenceDataResponse` schema does not list but the reference extract does? → A: Yes. Records include `name` (the human-readable title). This is an agreed, documented deviation from the Swagger schema (see Contract Deviations), justified because a title list without names has no practical use and the reference extract shows the real data carries it.
- Q: What credential scheme should the mock accept? → A: An `Authorization: Bearer <token>` header, where the token must exactly match one of a configured set of valid tokens. A missing header, a header not using the `Bearer` scheme, an empty token, or a token outside the configured set are all rejected with `401`.
- Q: May the fixture reuse title names from the supplied production-labelled extract? → A: Yes, title names only. The public judicial office titles may be reused as `name` values, but `id`, `created_at`, `updated_at`, `start_date` and `end_date` MUST be synthetic. This is consistent with Principle IV because the titles are public, non-personal job titles, not production records; no other column or value from the extract is carried over.
- Q: Should authentication be switchable off by configuration, or always required on reference-data routes? → A: Always required. There is no configuration switch to turn it off; only the set of accepted tokens is configurable.
- Q: Should an unsupported reference-data type (contract-valid but not built, e.g. `genders`, or wholly unknown, e.g. `foo`) return `400` or `404`? → A: `400 Bad Request` for both, because `attribute_name` is an enumerated parameter; `404` is reserved for an unknown `reference_id` under a supported type.
- Q: How many AppointmentTitle records should the default dataset contain? → A: One record for every distinct public title name in the supplied reference extract (194 titles), each with synthetic `id`, timestamps and dates.
- Q: Should undefined query parameters on reference-data requests be ignored or rejected? → A: Rejected with `400 Bad Request` in the shared error shape, since the contract defines no query parameters for reference data.
- Q: Should a trailing slash (e.g. `/api/v1/reference_data/appointment_titles/`) match the route without it, or be rejected? → A: Rejected. Routes match exactly; a trailing slash matches no route and returns `404 Not Found` in the shared error shape, with authentication still evaluated first.

## Behavioural Reference

The supplied E-Links Swagger (`swagger-ui-elinks-api-v5.pdf`, "Reference Data" section) defines, and this feature relies on, the following:

| Aspect | E-Links reference behaviour | This mock |
|--------|------------------------|----------------|
| Collection route | Get all records for `{attribute_name}` | `GET /api/v1/reference_data/{attribute_name}` |
| Single-record route | Get one record by `{reference_id}` | `GET /api/v1/reference_data/{attribute_name}/{reference_id}` |
| `attribute_name` | Required path string; enum of 11 canonical values, plus 11 deprecated singular aliases | Only `appointment_titles` and alias `appointment_title` supported in this feature |
| `reference_id` | Required path parameter, type `number` | Required, numeric |
| Collection 200 body | `ReferenceDataApiResponse` — `{ "results": [ ReferenceDataResponse, ... ] }` | Same |
| Single-record 200 body | A single `ReferenceDataResponse` object (not wrapped in `results`) | Same |
| `ReferenceDataResponse` fields | `id`, `updated_at`, `created_at`, `start_date`, `end_date` (all marked required) | Same, plus `name` (agreed deviation — see Contract Deviations) |
| Media type | `application/json` | Same |
| 401 | "Unauthorized. Invalid or missing token." | Same status |
| Pagination / filtering | None documented for reference data | None |
| Not-found, invalid attribute, malformed id, 5xx | **Not documented** | Governed by the project's shared error contract (see Assumptions) |

The mock exposes these routes under its own base path, `/api/v1/reference_data` (the project is versioned v1, matching `specs/001-healthcheck-api`). All other observable behaviour follows the reference above, except for the deviation listed below.

### Contract Deviations

Per the constitution's Governance section, intentional deviations from the E-Links contract are recorded here:

| # | Deviation | Reason | Type |
|---|-----------|--------|------|
| D-1 | Reference-data records include a `name` field (string, required), which is not in the Swagger `ReferenceDataResponse` schema | The supplied reference extract shows real AppointmentTitle data carries `name`, and without it consumers can't resolve an id to a title. Agreed in Clarifications, 2026-09-23. | Permanent, scope-justified; to be revisited if the real E-Links response is confirmed to omit `name` |

## User Scenarios & Testing *(mandatory)*

*Acceptance scenario and edge case IDs below (`AC-`/`EC-`) are stable identifiers intended to be referenced from plan.md, tasks.md, contracts/, and quickstart.md. Numbers AC-001–AC-014 correspond one-to-one to the fourteen acceptance scenarios in the feature request.*

### User Story 1 - Integrator Retrieves All Appointment Titles (Priority: P1)

As a developer or automated test suite integrating against the E-Links mock, I want to retrieve the complete list of appointment titles so that my system can load and resolve appointment-title reference data exactly as it would against the real E-Links API.

**Why this priority**: The collection route is the primary way consumers load reference data (typically once, to populate a lookup). Without it, no downstream integration involving appointment titles can be exercised. It is the MVP of this feature.

**Independent Test**: With valid credentials, call `GET /api/v1/reference_data/appointment_titles` and confirm a `200 OK` JSON response containing a `results` array of appointment-title records in the documented shape, identical on repeated calls.

**Acceptance Scenarios**:

1. **AC-001**: **Given** a caller with valid authentication, **When** they send `GET /api/v1/reference_data/appointment_titles`, **Then** the response is `200 OK` with content type `application/json` and a body of the form `{ "results": [ ... ] }` containing every AppointmentTitle record in the configured dataset, each in the reference-data record shape.
2. **AC-012**: **Given** the running service, **When** the Reference Data routes are inspected (via requests and via published API documentation), **Then** they are exposed under `/api/v1/reference_data` and no reference-data route is exposed under any other base path.
3. **AC-015**: **Given** the same configured dataset, **When** the collection route is called multiple times (including across service restarts), **Then** every response is byte-for-byte equivalent in content and record order.

---

### User Story 2 - Integrator Retrieves a Single Appointment Title by ID (Priority: P1)

As an integrator, I want to fetch one appointment title by its reference identifier so that my system can resolve an identifier it holds (for example, from a person's appointment) to its full reference record.

**Why this priority**: Equal in value to the collection route — both are part of the contract consumers depend on — and it exercises the key validation paths (malformed and unknown identifiers).

**Independent Test**: With valid credentials, call `GET /api/v1/reference_data/appointment_titles/{id}` for an `id` known to exist in the dataset and confirm a `200 OK` single-object response matching the corresponding record in the collection.

**Acceptance Scenarios**:

1. **AC-004**: **Given** valid authentication and an `id` present in the dataset, **When** the caller sends `GET /api/v1/reference_data/appointment_titles/{id}`, **Then** the response is `200 OK`, `application/json`, with a single reference-data record object (not wrapped in `results`) whose values equal the record with that `id` in the collection response.
2. **AC-008**: **Given** valid authentication, **When** the caller sends a request whose `reference_id` is not numeric (e.g. `abc`, `12x`, `1.5`, `-1`; a missing segment is covered by EC-004), **Then** the request is rejected with `400 Bad Request` in the shared error response shape, identically for the canonical and alias routes.
3. **AC-009**: **Given** valid authentication and a well-formed numeric `reference_id` that does not exist in the dataset, **When** the caller requests it, **Then** the response is `404 Not Found` in the shared error response shape, identically for the canonical and alias routes.
4. **AC-020**: **Given** valid authentication, **When** the caller sends a collection or single-record request carrying any query parameter (e.g. `GET /api/v1/reference_data/appointment_titles?name=Judge`), **Then** the request is rejected with `400 Bad Request` in the shared error response shape and no reference data is returned, identically for the canonical and alias routes.

---

### User Story 3 - Legacy Consumer Uses the Deprecated Alias (Priority: P2)

As a consumer still using the deprecated singular name `appointment_title`, I want my existing requests to keep working and return the same data as the canonical name, so that I am not forced to migrate before I am ready.

**Why this priority**: The E-Links contract explicitly supports deprecated aliases, so omitting them would break contract fidelity for legacy consumers; but the canonical name serves the majority of consumers, so this is secondary.

**Independent Test**: Call both the canonical and alias routes (collection and single-record) with the same credentials and compare responses.

**Acceptance Scenarios**:

1. **AC-002**: **Given** valid authentication, **When** the caller sends `GET /api/v1/reference_data/appointment_title`, **Then** the response is `200 OK`, `application/json`, `{ "results": [ ... ] }`.
2. **AC-003**: **Given** valid authentication, **When** the collection is fetched via `appointment_titles` and via `appointment_title`, **Then** the two response bodies are equal (same records, same field values, same order) and have the same status code and content type.
3. **AC-005**: **Given** valid authentication and an existing `id`, **When** the caller sends `GET /api/v1/reference_data/appointment_title/{id}`, **Then** the response is `200 OK` with the single record for that `id`.
4. **AC-006**: **Given** valid authentication, **When** the same `id` is fetched via `appointment_titles/{id}` and via `appointment_title/{id}`, **Then** the two responses are equal in status, content type, and body — for existing, unknown, and malformed ids alike.
5. **AC-016**: **Given** any request via the alias, **When** the response is returned, **Then** it contains no indication that a different attribute name was resolved (the alias is transparent to the caller), matching the E-Links reference, which documents no such signal.

---

### User Story 4 - Unsupported Reference-Data Types Are Rejected (Priority: P2)

As an integrator, I want requests for reference-data types this mock does not (yet) serve to be rejected clearly rather than returning empty or fabricated data, so that I can tell the difference between "no data" and "not supported".

**Why this priority**: Prevents consumers from gaining false confidence (Constitution Principle VI) and draws a clear line around this feature's scope.

**Independent Test**: Call the collection and single-record routes with an attribute name that is not `appointment_titles`/`appointment_title` and confirm rejection in the shared error shape.

**Acceptance Scenarios**:

1. **AC-007**: **Given** valid authentication, **When** the caller requests any attribute name other than `appointment_titles` or `appointment_title` — including names that are valid in the E-Links contract but out of scope for this feature (e.g. `genders`, `base_location`) and names that are entirely unknown (e.g. `foo`) — on either the collection or single-record route, **Then** the request is rejected with `400 Bad Request` in the shared error response shape, and no reference data is returned.
2. **AC-014**: **Given** the delivered feature, **When** each of the other 20 E-Links attribute names (the ten other canonical types and their ten deprecated aliases) is requested, **Then** every one is rejected per AC-007 — none returns `200`, including with an empty `results` array.

---

### User Story 5 - Unauthenticated or Invalidly Authenticated Requests Are Refused (Priority: P1)

As a security reviewer, I want reference-data requests without valid credentials to be refused with `401 Unauthorized`, consistently with the E-Links contract, so that consumers exercise their authentication handling against the mock just as they would against the real API.

**Why this priority**: The E-Links contract documents `401` for both reference-data operations; this is the first authenticated capability in the repository, so getting it right sets the precedent for every later feature.

**Independent Test**: Call each reference-data route with no credentials and with invalid credentials and confirm `401` in the shared error shape; confirm valid credentials succeed.

**Acceptance Scenarios**:

1. **AC-010**: **Given** a request with no authentication credentials, **When** it is sent to any reference-data route (canonical or alias, collection or single-record, supported or unsupported attribute, well-formed or malformed id), **Then** the response is `401 Unauthorized` in the shared error response shape and no reference data is returned.
2. **AC-011**: **Given** a request with credentials that are present but invalid (see FR-017), **When** it is sent to any reference-data route, **Then** the response is `401 Unauthorized` in the shared error response shape and no reference data is returned.
3. **AC-017**: **Given** the authentication behaviour is externally configured, **When** the configured valid credential(s) are changed, **Then** the service accepts the newly configured credential(s) and rejects the old ones, without any code change; no configuration value turns authentication off.
4. **AC-018**: **Given** authentication is introduced by this feature, **When** `GET /api/v1/healthcheck` is called without credentials, **Then** it still returns `200 OK` — the healthcheck's documented unauthenticated exemption (`specs/001-healthcheck-api`) is preserved and not broadened.

---

### User Story 6 - Maintainer Can Add Further Reference-Data Types Without Duplicating Endpoint Logic (Priority: P3)

As a maintainer, I want AppointmentTitle to be served by the shared, generic reference-data capability so that later features can add the remaining types (base_locations, contract_types, genders, judiciary_roles, jurisdictions, location_types, locations, ticket_categories, ticket_category_types, tickets) by supplying a type definition and its data, rather than new endpoints.

**Why this priority**: Delivers no immediate consumer-visible behaviour, but is a constitutional requirement (Principles III, VII) and determines the cost of the next ten features.

**Independent Test**: Review and automated tests confirm there is exactly one pair of reference-data routes (collection, single-record) parameterised by attribute name, a single central point where attribute names and deprecated aliases are resolved, and no AppointmentTitle-specific route, handler, validation, or error path.

**Acceptance Scenarios**:

1. **AC-013**: **Given** the delivered implementation, **When** it is reviewed and tested, **Then** AppointmentTitle is served through the generic reference-data capability — the attribute name is a route parameter resolved centrally (including the `appointment_title` alias), and there is no AppointmentTitle-specific endpoint behaviour.
2. **AC-019**: **Given** the delivered implementation, **When** a maintainer follows the documented extension approach (to be defined in plan.md) to register a new reference-data type in a test, **Then** that type becomes retrievable through the same routes, with the same validation, authentication, and error behaviour, without adding or modifying endpoint logic.

---

### Edge Cases

- **EC-001 — Attribute name case**: `GET /api/v1/reference_data/Appointment_Titles` or `APPOINTMENT_TITLES` is treated as an unsupported attribute (AC-007). Attribute names are matched exactly and case-sensitively, as enumerated in the contract.
- **EC-002 — Trailing slash**: Routes match exactly. `GET /api/v1/reference_data/appointment_titles/` and `GET /api/v1/reference_data/appointment_titles/1/` match no route and return `404 Not Found` in the shared error shape; they are neither treated as equivalent to the slash-less route nor redirected.
- **EC-003 — Leading zeros / large ids**: A `reference_id` of `007` is numeric and is parsed as id `7`, then looked up like any other id (so `010` returns record `10`, and `007` returns `404` if no record `7` exists). A numeric `reference_id` too large to be a valid identifier is treated as unknown (`404`) — not as a server error.
- **EC-004 — Missing reference_id segment**: `GET /api/v1/reference_data/appointment_titles/` with no id is not a single-record request; it returns `404` per EC-002.
- **EC-005 — Extra path segments**: `GET /api/v1/reference_data/appointment_titles/1/extra` matches no reference-data route and receives the shared `404` response.
- **EC-013 — Authentication on unmatched reference-data paths**: Any request under `/api/v1/reference_data/` is authenticated first, so an unauthenticated request to an unmatched path (EC-002, EC-004, EC-005) receives `401`, not `404` (consistent with EC-009).
- **EC-006 — Unexpected query parameters**: The contract defines no query parameters for reference data. Any query parameter on a reference-data request (e.g. `?page=2`, `?name=Judge`, even with an empty value) is rejected with `400 Bad Request` in the shared error shape (FR-027), so no parameter is ever accepted and silently ignored (Principle VI).
- **EC-007 — Unsupported methods**: `POST`, `PUT`, `PATCH`, `DELETE` on reference-data routes are rejected with `405 Method Not Allowed` in the shared error shape (reference data is read-only).
- **EC-008 — Unacceptable media type**: A request whose `Accept` header excludes `application/json` receives the application's standard not-acceptable response; the contract documents only `application/json`.
- **EC-009 — Evaluation order**: An unauthenticated request for an unsupported attribute, a malformed id or an undefined query parameter returns `401`, not `400`, because authentication is evaluated first (FR-019). Unauthenticated callers therefore cannot probe which types are supported.
- **EC-010 — Unexpected internal failure**: If an unexpected error occurs while serving a reference-data request (e.g. a fault while mapping or serialising a record), the response is `500 Internal Server Error` in the shared error shape, with no stack trace, internal class names, file paths, or dataset contents exposed. A fixture that fails validation is not a runtime error: it stops the application from starting, so no request is ever served from invalid data.
- **EC-011 — Empty configured dataset**: If the AppointmentTitle dataset is configured to be empty, the collection route returns `200` with `{ "results": [] }` and every single-record request returns `404`. (This is distinct from an *unsupported* type, which returns `400`.)
- **EC-012 — Null end dates**: Records with no end date (currently active titles) return `end_date` as JSON `null`; the field is always present.

## Requirements *(mandatory)*

### Functional Requirements

**Routing and scope**

- **FR-001**: System MUST expose `GET /api/v1/reference_data/{attribute_name}` returning all records of the reference-data type identified by `attribute_name`.
- **FR-002**: System MUST expose `GET /api/v1/reference_data/{attribute_name}/{reference_id}` returning the single record of that type whose `id` equals `reference_id`.
- **FR-003**: In this feature the only accepted `attribute_name` values MUST be `appointment_titles` (canonical) and `appointment_title` (deprecated alias), both resolving to the AppointmentTitle reference-data type.
- **FR-004**: Canonical and alias forms MUST produce identical responses (status, content type, body, record order) for every request, including success, not-found, and validation failures.
- **FR-005**: Resolution of attribute names — including deprecated aliases — MUST happen in exactly one shared place, used by both routes; neither route may contain type- or alias-specific logic.
- **FR-006**: Reference-data routes MUST be read-only and MUST NOT modify any application state.
- **FR-007**: System MUST NOT serve any reference-data type other than AppointmentTitle, and MUST NOT expose placeholder or empty responses for the other ten E-Links types.
- **FR-008**: The reference-data capability MUST be structured so that a future feature can add one of the ten remaining types (with its canonical name and deprecated alias) by supplying its type definition and dataset, without adding or duplicating route, validation, authentication, or error-handling logic.

**Response content**

- **FR-009**: A successful collection response MUST be `200 OK`, `application/json`, with a body containing a single top-level field `results`, whose value is an array of reference-data records (the E-Links `ReferenceDataApiResponse` shape).
- **FR-010**: A successful single-record response MUST be `200 OK`, `application/json`, with a body that is a single reference-data record object, not wrapped in `results` (the E-Links `ReferenceDataResponse` shape).
- **FR-011**: Each reference-data record MUST contain the fields defined by the E-Links `ReferenceDataResponse` schema: `id` (integer), `created_at` (ISO-8601 UTC date-time, e.g. `2023-04-13T09:28:20Z`), `updated_at` (same format), `start_date` (ISO-8601 date, e.g. `2023-03-30`), and `end_date` (ISO-8601 date, or `null` when the record has no end date), plus `name` (non-empty string; the human-readable title, e.g. "Circuit Judge") per deviation D-1.
- **FR-012**: Records in a collection response MUST be returned in ascending `id` order.
- **FR-013**: The response MUST NOT contain fields beyond those confirmed in FR-009–FR-011 (no pagination metadata, links, counts, type names, or alias indicators).

**Validation**

- **FR-014**: An `attribute_name` that is not a supported value (FR-003) MUST be rejected with `400 Bad Request` in the shared error response shape, on both routes, and MUST NOT be silently accepted or mapped to another type. This applies equally to values valid in the E-Links contract but outside this feature's scope; `404` MUST NOT be used for an unsupported `attribute_name`.
- **FR-015**: `reference_id` MUST be a non-negative whole number expressed in decimal digits only. Any other value (letters, decimals, signs, whitespace, symbols) MUST be rejected with `400 Bad Request` in the shared error response shape.
- **FR-016**: A well-formed `reference_id` that matches no record MUST return `404 Not Found` in the shared error response shape.
- **FR-027**: Reference-data routes accept no query parameters. A request carrying any query parameter MUST be rejected with `400 Bad Request` in the shared error response shape; query parameters MUST NOT be silently ignored.

**Authentication**

- **FR-017**: All reference-data routes MUST require authentication, enforced by one centralised, externally configurable mechanism (not per-endpoint logic), whose accepted credential(s) are defined by configuration rather than code. Credentials MUST be supplied as an `Authorization: Bearer <token>` header (scheme name case-insensitive; see Assumptions); a request is authenticated only if the token exactly matches one of a configured set of valid tokens. A missing `Authorization` header counts as missing authentication; a non-`Bearer` scheme, an empty token, or a token outside the configured set counts as invalid authentication. Authentication MUST NOT be disableable by configuration: only the set of accepted tokens is configurable, and no setting (including an empty token set) may cause reference-data routes to be served without a valid token. Configured tokens MUST NOT be written to logs or error responses.
- **FR-018**: Missing and invalid authentication, as defined in FR-017, MUST both receive `401 Unauthorized` in the shared error response shape, with the same generic message, so the response does not reveal which case applied.
- **FR-019**: Authentication MUST be evaluated before attribute-name, `reference_id` and query-parameter validation (EC-009).
- **FR-020**: Introducing authentication MUST NOT change the unauthenticated behaviour of `GET /api/v1/healthcheck` (AC-018), and MUST NOT create any other unauthenticated business endpoint.

**Data**

- **FR-021**: The AppointmentTitle dataset MUST be synthetic, deterministic, and supplied via fixed, externally configurable fixtures; the same request against the same configured dataset MUST return the same response. Title `name` values MAY be taken from the public judicial office titles in the supplied reference extract; every other value (`id`, `created_at`, `updated_at`, `start_date`, `end_date`) MUST be synthetic, and no other data from the extract may be carried over. Each `name` MUST be unique within the dataset. The default dataset MUST contain exactly one record for each of the 194 distinct title names in the supplied AppointmentTitle reference extract, no more and no fewer.
- **FR-022**: Every AppointmentTitle record MUST have a unique `id`; within each record `created_at` MUST NOT be later than `updated_at`, and where `end_date` is present it MUST NOT be earlier than `start_date`.
- **FR-023**: The default dataset MUST include at least one record with a `null` `end_date` and at least one with a non-null `end_date`, and MUST contain gaps in the `id` sequence, so that consumers and tests can exercise these cases.
- **FR-024**: AppointmentTitle `id` values MUST be stable across releases of the default dataset, so they can be safely referenced by later features (e.g. People API appointments) and by consumer test suites.

**Errors**

- **FR-025**: Every error response from reference-data routes (`400`, `401`, `404`, `405`, `406`, `500`) MUST use the project's single shared error response shape; no reference-data-specific error body may be introduced.
- **FR-026**: An unexpected internal failure MUST produce `500 Internal Server Error` in the shared error shape, exposing no internal detail (EC-010).

### Error Scenario Summary

| Scenario | Example request | Expected result |
|----------|-----------------|-----------------|
| Successful collection (canonical) | `GET /api/v1/reference_data/appointment_titles` | `200`, `{ "results": [...] }` |
| Successful collection (alias) | `GET /api/v1/reference_data/appointment_title` | `200`, identical to canonical |
| Successful single record (canonical) | `GET /api/v1/reference_data/appointment_titles/1` | `200`, single record |
| Successful single record (alias) | `GET /api/v1/reference_data/appointment_title/1` | `200`, identical to canonical |
| Unsupported type (contract-valid, out of scope) | `GET /api/v1/reference_data/genders` | `400`, shared error shape |
| Unsupported type (unknown) | `GET /api/v1/reference_data/foo/1` | `400`, shared error shape |
| Malformed `reference_id` | `GET /api/v1/reference_data/appointment_titles/abc` | `400`, shared error shape |
| Undefined query parameter | `GET /api/v1/reference_data/appointment_titles?page=2` | `400`, shared error shape |
| Unknown `reference_id` | `GET /api/v1/reference_data/appointment_titles/999999` | `404`, shared error shape |
| Missing authentication | any route, no credentials | `401`, shared error shape |
| Invalid authentication | any route, wrong credentials | `401`, shared error shape |
| Unsupported method | `POST /api/v1/reference_data/appointment_titles` | `405`, shared error shape |
| Unacceptable media type | `GET /api/v1/reference_data/appointment_titles` with `Accept: text/plain` | `406`, shared error shape (sent as `application/json`) |
| Trailing slash | `GET /api/v1/reference_data/appointment_titles/` | `404`, shared error shape |
| Unexpected internal failure | any route, dataset fault | `500`, shared error shape, no internals |

### Key Entities

- **Reference-data type**: A named category of reference data (here, only AppointmentTitle). Identified by exactly one canonical attribute name (`appointment_titles`) and zero or more deprecated aliases (`appointment_title`). Future types follow the same model.
- **AppointmentTitle record**: One judicial appointment title. Attributes: `id` (unique, stable integer identifier), `created_at`, `updated_at` (UTC timestamps), `start_date`, `end_date` (dates; `end_date` optional), and `name` (unique, human-readable title; deviation D-1). Will later be referenced by People-API appointment data; its ids must therefore remain internally consistent with any future dataset that refers to them.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% of the 20 acceptance scenarios (AC-001–AC-020) pass as automated tests.
- **SC-002**: For every request pair differing only by `appointment_titles` vs `appointment_title`, responses are identical in 100% of tested cases (success, not-found, malformed id, undefined query parameter).
- **SC-003**: 100 consecutive identical requests against the same configured dataset — including across a service restart — return identical responses.
- **SC-004**: 0 of the 20 other E-Links attribute names (10 canonical + 10 deprecated aliases) return a `200` response.
- **SC-005**: 0 reference-data requests without valid credentials receive any reference data; 100% receive `401`.
- **SC-006**: An integrator using the published API documentation and the example requests can retrieve the full appointment-title list and a single title on their first attempt, without reading source code.
- **SC-007**: Adding a second reference-data type in a later feature requires no new or modified route, validation, authentication, or error-handling logic (verified at that feature's review by AC-019's approach).

## Assumptions

- **Status codes the contract omits**: The E-Links Swagger documents only `200` and `401` for reference data. The following codes are this project's reasonable defaults under its shared error contract (Constitution Principle IX), not observed E-Links behaviour: `400` for unsupported `attribute_name`, malformed `reference_id` and any query parameter, `404` for unknown `reference_id` and for unmatched paths such as a trailing slash, `405` for unsupported methods, `500` for unexpected failures. `400` (rather than `404`) for unsupported attribute names was confirmed in Clarifications (2026-09-23). The remaining defaults may be revisited during `/speckit-clarify` if real E-Links behaviour becomes known.
- **Numeric `reference_id`**: The contract types `reference_id` as `number`; all known identifiers are whole numbers, so fractional and negative values are treated as malformed.
- **No pagination or filtering**: The contract defines none for reference data; the collection returns all records in one response. The default AppointmentTitle set is 194 records (FR-021), small enough to return in one response.
- **Nullable `end_date`**: The schema marks `end_date` required, and the reference extract leaves it blank for active titles; the field is therefore always present and `null` when there is no end date.
- **Timestamps** are rendered in UTC with a `Z` suffix and whole seconds, matching the contract example.
- **Bearer scheme case**: The `Bearer` scheme name in the `Authorization` header is matched case-insensitively (so `bearer <token>` is accepted), following RFC 7235's rule that authentication scheme names are case-insensitive. The token itself is compared exactly, and is case-sensitive.
- **Alias transparency**: The contract documents no deprecation header or response flag, so none is added.
- **Case sensitivity**: Attribute names are matched exactly as enumerated in the contract.
- **Authentication introduced here**: No authentication mechanism exists in the repository yet; this feature introduces the shared mechanism that later business endpoints will reuse. The healthcheck exemption from `specs/001-healthcheck-api` remains unchanged.
- **Base path**: All reference-data routes are served under `/api/v1/reference_data`, as specified for this project.
- **Test and local tokens**: The default configuration ships a clearly non-secret token for local runs, tests and the Postman environment; real deployments are expected to override it via configuration.
- **Dependencies**: Relies on the shared error response shape, global error handling, and correlation-ID handling required by the constitution (some of which may be first built in this feature if not already present from `001-healthcheck-api`).
