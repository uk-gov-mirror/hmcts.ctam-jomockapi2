# Quickstart: Reference Data API — Appointment Titles

**Feature**: `002-reference-data-api` | **Contract**: [contracts/reference-data-api.md](./contracts/reference-data-api.md) | **Data model**: [data-model.md](./data-model.md)

This guide runs and checks the feature end to end. For expected response shapes and messages, see the contract; they are not repeated here.

## Prerequisites

- JDK 25 on the `PATH`, or found through the Gradle toolchain. Check with `java -version`.
- Optional: Node.js, to run the Postman collection with `npx newman`.
- Optional: an NVD API key in `NVD_API_KEY`, to run the OWASP dependency check locally.

## 1. Build with every quality gate

```bash
./gradlew check                 # compile (-Werror), Checkstyle, the four test suites, JaCoCo, OWASP check
./gradlew check -PskipOwasp     # quicker local run without the NVD download (CI never skips it)
./gradlew dependencyUpdates     # dependency freshness report
```

**Expected**: `BUILD SUCCESSFUL`. The coverage report is at `build/reports/jacoco/test/html/index.html`.

Run one suite at a time with `./gradlew test`, `integrationTest`, `functionalTest` or `smokeTest`.

## 2. Run the service

```bash
./gradlew bootRun
```

The default local token is the non-secret `local-dev-token`. To use a different one:

```bash
JO_SECURITY_BEARER_TOKENS=my-token ./gradlew bootRun           # one token
JO_SECURITY_BEARER_TOKENS=token-a,token-b ./gradlew bootRun    # several tokens, comma-separated
```

This works because `application.yml` refers to `${JO_SECURITY_BEARER_TOKENS:local-dev-token}` explicitly (research R2).

If you set `JO_SECURITY_BEARER_TOKENS` to an empty value, the service **should refuse to start** (research R2).

## 3. Manual checks with curl

```bash
BASE=http://localhost:8080/api/v1/reference_data
AUTH='Authorization: Bearer local-dev-token'
```

| # | Command | Expected (details in the contract) | Spec |
|---|---------|------------------------------------|------|
| 1 | `curl -s -H "$AUTH" $BASE/appointment_titles \| jq '.results \| length'` | `194` | AC-001 |
| 2 | `diff <(curl -s -H "$AUTH" $BASE/appointment_titles) <(curl -s -H "$AUTH" $BASE/appointment_title)` | No differences | AC-002, AC-003 |
| 3 | `curl -s -H "$AUTH" $BASE/appointment_titles/70` | The `Area Coroner` record, with `end_date` `2025-03-31` | AC-004 |
| 4 | `diff <(curl -s -H "$AUTH" $BASE/appointment_titles/10) <(curl -s -H "$AUTH" $BASE/appointment_title/10)` | No differences | AC-005, AC-006 |
| 5 | `curl -si -H "$AUTH" $BASE/genders` | `400`, unsupported attribute message | AC-007 |
| 6 | `curl -si -H "$AUTH" $BASE/appointment_titles/abc` | `400`, malformed id message | AC-008 |
| 7 | `curl -si -H "$AUTH" $BASE/appointment_titles/15` | `404`, record not found | AC-009 |
| 8 | `curl -si $BASE/appointment_titles` | `401` and `WWW-Authenticate: Bearer` | AC-010 |
| 9 | `curl -si -H 'Authorization: Bearer wrong' $BASE/appointment_titles` | `401` | AC-011 |
| 10 | `curl -si -H "$AUTH" "$BASE/appointment_titles?page=2"` | `400`, query parameter message | AC-020 |
| 11 | `curl -si -H "$AUTH" $BASE/appointment_titles/` | `404`, resource not found | EC-002 |
| 12 | `curl -si -H "$AUTH" -X POST $BASE/appointment_titles` | `405` in the shared error shape | EC-007 |
| 13 | `curl -si -H "$AUTH" -H 'X-Correlation-Id: demo-123' $BASE/foo` | `400`, `traceId` = `demo-123`, and the response header echoes it | Principle XIV |
| 14 | `curl -s http://localhost:8080/api/v1/healthcheck` | `{"status":"ok"}` with no token | AC-018 |
| 15 | `curl -si --path-as-is http://localhost:8080/%61pi/v1/reference_data/appointment_titles` | `401` (or a container `400`), **never** `200` | Principle XV (encoded-path bypass) |
| 16 | `curl -si -H "$AUTH" -H 'Accept: text/plain' $BASE/appointment_titles` | `406`, JSON error body | EC-008 |

## 4. OpenAPI

- `http://localhost:8080/v3/api-docs`: both reference-data paths, the `bearerAuth` scheme, the `attribute_name` enum (`appointment_titles`, `appointment_title`), and `200`/`400`/`401`/`404`/`406`/`500` responses (`404` on the single-record route only) using the three schemas in the contract.
- `http://localhost:8080/swagger-ui.html`: use **Authorize** with `local-dev-token`, then try both operations (SC-006).

## 5. Postman

```bash
npx newman run postman/ctam-jomockapi.postman_collection.json \
  -e postman/ctam-jomockapi.postman_environment.json
```

**Expected**: every request in the Healthcheck and Reference Data folders passes its tests. The environment supplies `baseUrl` and `bearerToken` (`local-dev-token`).

## 6. Logs

While the service is running, send request 13 above. The console shows one JSON log line containing `"correlationId":"demo-123"` at `WARN` level for the `400`. No log line contains a token value.
