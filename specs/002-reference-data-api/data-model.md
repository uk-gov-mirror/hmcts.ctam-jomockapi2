# Data Model: Reference Data API — Appointment Titles

**Feature**: `002-reference-data-api` | **Date**: 2026-09-23 | **Spec**: [spec.md](./spec.md) | **Research**: [research.md](./research.md)

The model has three layers:

1. **Configuration**: which reference-data types exist.
2. **Internal records**: loaded from fixture files.
3. **API DTOs**: what callers receive.

MapStruct maps internal records to DTOs (research R9). Class names below are the planned names, and package paths are relative to `uk.gov.hmcts.ctam.jo`.

---

## 1. ReferenceDataType (configuration → `entity/ReferenceDataType`)

This is one declared reference-data type. The types come from `jo.reference-data.types[*]` (bound by `config/ReferenceDataProperties`). `services/ReferenceDataTypeRegistry` turns them into immutable values at startup.

| Field | Type | Rules |
|-------|------|-------|
| `name` | String | The canonical attribute name, e.g. `appointment_titles`. Required, non-blank, and matching `^[a-z][a-z_]*$`. |
| `aliases` | Set<String> | The deprecated attribute names, e.g. `{appointment_title}`. It may be empty. Each alias follows the same pattern as `name`. |
| `fixture` | Resource location | Where the records are, e.g. `classpath:reference-data/appointment_titles.json`. Required, and the resource must exist. |

**Registry rules** (checked at startup; any breach stops the application from starting):
- All names and aliases across all types are unique. No string is both one type's name and another type's alias, and no type lists its own name as an alias.
- Lookups match exactly and are case-sensitive (spec EC-001).
- `resolve(String attributeName)` returns the type for a canonical name or an alias. For anything else it throws `UnsupportedReferenceDataTypeException`, which becomes `400` (FR-014).
- This is the only place in the code where aliases are resolved (FR-005).

**Shipped configuration**: exactly one type, `appointment_titles` with the alias `appointment_title` (FR-003, FR-007).

---

## 2. ReferenceDataRecord (fixture → `entity/ReferenceDataRecord`)

This is one reference-data record as stored in a fixture file. The fixture JSON uses the same snake_case field names as the API.

| Field | Java type | JSON (fixture) | Rules |
|-------|-----------|----------------|-------|
| `id` | `long` | `id` (number) | Required, > 0, unique within the type (FR-022) |
| `name` | `String` | `name` (string) | Required, non-blank, trimmed, unique within the type (FR-021; deviation D-1) |
| `createdAt` | `Instant` | `created_at` (`yyyy-MM-ddTHH:mm:ssZ`) | Required, UTC, whole seconds |
| `updatedAt` | `Instant` | `updated_at` | Required, and not before `createdAt` (FR-022) |
| `startDate` | `LocalDate` | `start_date` (`yyyy-MM-dd`) | Required |
| `endDate` | `LocalDate` (nullable) | `end_date` (`yyyy-MM-dd` or `null`) | Optional. If set, not before `startDate` (FR-022) |

**Fixture file format**: a top-level JSON array of record objects. The records don't have to be in order, because the repository sorts them by `id`.

**Load-time validation** (`repository/FixtureReferenceDataRepository`, at startup):
- All the field rules above are checked.
- Unknown JSON properties are rejected, so typos surface.
- Any breach stops startup with a message naming the fixture and the record `id`. The data is never partly served.

**State**: none. Records are immutable and read-only (FR-006). There are no lifecycle transitions.

**Relationships**: none in this feature. People-API appointments will refer to `id` later, which is why ids must stay stable (FR-024).

---

## 3. API DTOs (`domain/`)

### ReferenceDataResponse

This matches the E-Links `ReferenceDataResponse`, plus `name`. It is returned as-is by the single-record route (FR-010) and appears as each element of `results` (FR-009).

| JSON field | Java component | JSON type | Nullable | Source |
|------------|----------------|-----------|----------|--------|
| `id` | `long id` | integer | no | `ReferenceDataRecord.id` |
| `name` | `String name` | string | no | `ReferenceDataRecord.name` (deviation D-1) |
| `created_at` | `Instant createdAt` | string, `date-time` | no | `ReferenceDataRecord.createdAt` |
| `updated_at` | `Instant updatedAt` | string, `date-time` | no | `ReferenceDataRecord.updatedAt` |
| `start_date` | `LocalDate startDate` | string, `date` | no | `ReferenceDataRecord.startDate` |
| `end_date` | `LocalDate endDate` | string, `date` | **yes**, and the field is always present | `ReferenceDataRecord.endDate` |

Field order in the JSON is `id`, `name`, `created_at`, `updated_at`, `start_date`, `end_date`, fixed with `@JsonPropertyOrder` so output is deterministic. No other fields are allowed (FR-013).

### ReferenceDataApiResponse

| JSON field | Java component | Rules |
|------------|----------------|-------|
| `results` | `List<ReferenceDataResponse> results` | Every record of the resolved type, in ascending `id` order (FR-012). `[]` if the configured fixture is empty (EC-011). |

### ErrorResponse (`exceptions/ErrorResponse`)

This is the one shared error body (Principle IX; spec FR-025). The contract lists the messages used.

| JSON field | Java component | Rules |
|------------|----------------|-------|
| `error` | `String error` | A fixed, generic, human-readable message. It never echoes caller input or token values. |
| `timestamp` | `Instant timestamp` | From the injected `Clock`, in UTC ISO-8601. |
| `traceId` | `String traceId` | The request's correlation ID (the `X-Correlation-Id` value). `null` only on the exempt healthcheck path. |

---

## 4. Mapping (`mappers/ReferenceDataMapper`)

This is a MapStruct interface with `componentModel = "spring"`:

- `ReferenceDataResponse toResponse(ReferenceDataRecord record)`: a field-for-field mapping.
- `List<ReferenceDataResponse> toResponses(List<ReferenceDataRecord> records)`: keeps the input order.

The service wraps the list in `ReferenceDataApiResponse`. The mapper has no logic of its own, because each field has the same name and type on both sides.

---

## Fixture generation rules

`src/main/resources/reference-data/appointment_titles.json` is generated **once** during implementation and then checked in. After that, the checked-in file is the source of truth; it is not regenerated during builds.

1. **Input**: the `name` column of `eLinks_Pivotl_Production_all-data_2026-06-01_REF_AppointmentTitle.csv` (in the sibling checkout `../ctam-jomockapi/joh-elinks-api/ReferenceData/`). It holds 194 rows, all with distinct names. Every other column is ignored (FR-021).
2. **Order**: names are sorted by Java `String` natural (code-point) order. All 194 names are ASCII and trimmed, so this order is unambiguous. Positions run from 1 to 194.
3. **`id`**: `10 × position`, giving 10, 20, … 1940. That leaves gaps between every pair (FR-023). The numbers may coincide with some of the extract's own ids, but they are assigned by this rule, not copied from the extract.
4. **Records where `position % 7 == 0`** (27 records): `end_date = 2025-03-31` and `updated_at = 2025-04-01T08:00:00Z`.
5. **All other records** (167): `end_date = null` and `updated_at = 2024-06-03T10:30:00Z`.
6. **Every record**: `created_at = 2024-01-15T09:00:00Z` and `start_date = 2024-01-01`.

Every value meets the FR-022 rules: `created_at ≤ updated_at`, and `end_date ≥ start_date`.

**Known reference points** (used by tests and Postman):

| Position | `id` | `name` | `end_date` |
|----------|------|--------|------------|
| 1 | 10 | Acting Senior Coroner | null |
| 7 | 70 | Area Coroner | 2025-03-31 |
| 194 | 1940 | Vice-President, Employment Tribunal (Scotland) | null |

`id` 15 is a guaranteed unknown id inside the range, and `999999` is one outside it.

**Adding titles later**: give each new title the next unused multiple of 10 after the current highest `id`. Never renumber existing records (FR-024).
