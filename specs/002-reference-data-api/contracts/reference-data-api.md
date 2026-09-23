# Contract: Reference Data API (v1)

**Feature**: `002-reference-data-api` | **Spec**: [../spec.md](../spec.md) | **Data model**: [../data-model.md](../data-model.md)

This is the externally observable HTTP contract. The springdoc-generated `/v3/api-docs` MUST describe exactly what is here (Principle XVIII), and contract tests assert the two match.

---

## Common to every reference-data request

| Aspect | Contract |
|--------|----------|
| Base path | `/api/v1/reference_data` |
| Methods | `GET` only. `POST`, `PUT`, `PATCH` and `DELETE` on a matched route return `405`. `HEAD` and `OPTIONS` follow Spring MVC defaults (see Notes). |
| Authentication | `Authorization: Bearer <token>` is required. The token must be in `jo.security.bearer-tokens`. OpenAPI security scheme: `bearerAuth` (HTTP, `bearer`). |
| Correlation | `X-Correlation-Id` request header, optional. If it matches `^[A-Za-z0-9._-]{1,64}$` it is used; otherwise a UUID is generated. The value is always echoed in the `X-Correlation-Id` response header and in `traceId` on errors. |
| Query parameters | None are allowed. Any query parameter returns `400`. |
| Response media type | `application/json` |
| Path matching | Exact match only. A trailing slash or extra segments return `404`. |

**Order of checks** (the first failure decides the response):

1. authentication (`401`)
2. route and method (`404` / `405`)
3. query parameters (`400`)
4. `attribute_name` (`400`)
5. `reference_id` format (`400`)
6. record lookup (`404`)

Anything unexpected returns `500`.

---

## `GET /api/v1/reference_data/{attribute_name}`

OpenAPI: `operationId: getReferenceData`, summary "Get reference data", tag `Reference Data`.

**Path parameters**

| Name | Type | Required | Allowed values |
|------|------|----------|----------------|
| `attribute_name` | string | yes | `appointment_titles`, or `appointment_title` (deprecated alias). The OpenAPI `enum` is filled from the configured types. |

**Header parameters**

| Name | Type | Required | Rules |
|------|------|----------|-------|
| `Authorization` | string | yes | `Bearer <token>`. Documented through the `bearerAuth` security scheme, not as a parameter. |
| `X-Correlation-Id` | string, documented with `pattern: ^[A-Za-z0-9._-]{1,64}$` | no | Used if it matches the pattern; otherwise a UUID is generated. |

**Response headers**

| Name | On | Value |
|------|----|-------|
| `X-Correlation-Id` | every response | The correlation ID in use (the caller's, if valid, or the generated one). Equal to `traceId` on error bodies. |
| `WWW-Authenticate` | `401` only | `Bearer` |

**Responses**

| Status | Body | Notes |
|--------|------|-------|
| `200` | `ReferenceDataApiResponse` | `{ "results": [ ReferenceDataResponse… ] }`, in ascending `id` order |
| `400` | `ErrorResponse` | Unsupported `attribute_name`, or any query parameter |
| `401` | `ErrorResponse` | Missing or invalid token. Also sets `WWW-Authenticate: Bearer`. |
| `406` | `ErrorResponse` | `Accept` header excludes `application/json`. The body is still sent as `application/json`. |
| `500` | `ErrorResponse` | Unexpected failure |

**Example `200`** (truncated to the first record; the full default response has 194):

```json
{
  "results": [
    {
      "id": 10,
      "name": "Acting Senior Coroner",
      "created_at": "2024-01-15T09:00:00Z",
      "updated_at": "2024-06-03T10:30:00Z",
      "start_date": "2024-01-01",
      "end_date": null
    }
  ]
}
```

---

## `GET /api/v1/reference_data/{attribute_name}/{reference_id}`

OpenAPI: `operationId: getReferenceDataById`, summary "Get reference data by id", tag `Reference Data`.

**Path parameters**

| Name | Type | Required | Rules |
|------|------|----------|-------|
| `attribute_name` | string | yes | Same as above |
| `reference_id` | string, documented with `pattern: ^[0-9]+$` | yes | Decimal digits only. Leading zeros are allowed (`007` means 7). |

**Header parameters**

| Name | Type | Required | Rules |
|------|------|----------|-------|
| `Authorization` | string | yes | `Bearer <token>`. Documented through the `bearerAuth` security scheme, not as a parameter. |
| `X-Correlation-Id` | string, documented with `pattern: ^[A-Za-z0-9._-]{1,64}$` | no | Used if it matches the pattern; otherwise a UUID is generated. |

**Response headers**

| Name | On | Value |
|------|----|-------|
| `X-Correlation-Id` | every response | The correlation ID in use (the caller's, if valid, or the generated one). Equal to `traceId` on error bodies. |
| `WWW-Authenticate` | `401` only | `Bearer` |

**Responses**

| Status | Body | Notes |
|--------|------|-------|
| `200` | `ReferenceDataResponse` | A single object, not wrapped in `results` |
| `400` | `ErrorResponse` | Unsupported `attribute_name`, malformed `reference_id`, or any query parameter |
| `401` | `ErrorResponse` | Missing or invalid token. Also sets `WWW-Authenticate: Bearer`. |
| `404` | `ErrorResponse` | `reference_id` is well formed but has no record, including values too large to be an id |
| `406` | `ErrorResponse` | `Accept` header excludes `application/json`. The body is still sent as `application/json`. |
| `500` | `ErrorResponse` | Unexpected failure |

**Example `200`** (`GET /api/v1/reference_data/appointment_titles/70`):

```json
{
  "id": 70,
  "name": "Area Coroner",
  "created_at": "2024-01-15T09:00:00Z",
  "updated_at": "2025-04-01T08:00:00Z",
  "start_date": "2024-01-01",
  "end_date": "2025-03-31"
}
```

---

## Schemas

### `ReferenceDataResponse`

| Field | Type | Format | Required | Nullable | Description |
|-------|------|--------|----------|----------|-------------|
| `id` | integer | int64 | yes | no | Stable reference-data identifier |
| `name` | string | — | yes | no | Human-readable title (deviation D-1 from the E-Links schema) |
| `created_at` | string | date-time | yes | no | When the record was created (UTC) |
| `updated_at` | string | date-time | yes | no | When the record was last updated (UTC) |
| `start_date` | string | date | yes | no | Date from which the record is valid |
| `end_date` | string | date | yes (always present) | yes | Date the record stopped being valid. `null` if it is still current. |

No other properties are allowed.

### `ReferenceDataApiResponse`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `results` | array of `ReferenceDataResponse` | yes | Every record of the requested type |

### `ErrorResponse`

| Field | Type | Format | Required | Nullable | Description |
|-------|------|--------|----------|----------|-------------|
| `error` | string | — | yes | no | Human-readable error message |
| `timestamp` | string | date-time | yes | no | When the error was produced (UTC) |
| `traceId` | string | — | yes | yes (only on the exempt healthcheck path) | The request's correlation ID |

---

## Error messages

The messages are fixed, so they are deterministic and never echo caller input. Tests assert them exactly.

| Status | Cause | `error` |
|--------|-------|---------|
| `400` | Unsupported `attribute_name` | `Unsupported reference data attribute_name.` |
| `400` | Malformed `reference_id` | `reference_id must be a non-negative whole number.` |
| `400` | Any query parameter | `Query parameters are not supported on this endpoint.` |
| `401` | Missing or invalid token | `Unauthorized. Invalid or missing token.` |
| `404` | Unknown `reference_id` | `Reference data record not found.` |
| `404` | No matching route (e.g. trailing slash, extra segment) | `Resource not found.` |
| `405` | Method not allowed | `Method not allowed.` |
| `406` | `Accept` excludes `application/json` | `Not acceptable.` |
| `500` | Unexpected failure | `Internal server error.` |

**Example `401`**:

```http
HTTP/1.1 401 Unauthorized
Content-Type: application/json
WWW-Authenticate: Bearer
X-Correlation-Id: 3f1c2a9e-8b7d-4e21-9a55-0c6d1e2f3a4b

{"error":"Unauthorized. Invalid or missing token.","timestamp":"2026-09-23T10:15:30Z","traceId":"3f1c2a9e-8b7d-4e21-9a55-0c6d1e2f3a4b"}
```

---

## Undocumented, and deliberately so

- `/v3/api-docs` and `/swagger-ui/**` are served without authentication. They sit outside `/api/` (plan.md, Principle XV).
- `HEAD` and `OPTIONS` follow Spring MVC's defaults for `GET` routes. They are not part of this contract and are not asserted.
