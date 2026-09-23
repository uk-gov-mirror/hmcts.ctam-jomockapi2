# Research: Reference Data API — Appointment Titles

**Feature**: `002-reference-data-api` | **Date**: 2026-09-23 | **Spec**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md)

Each entry follows *Decision / Rationale / Alternatives considered*. Versions were checked against Maven Central and the Gradle Plugin Portal on 2026-09-23. The first Setup task confirms they resolve and build under the repo's Gradle 9.7.1 wrapper and Java 25 toolchain, then writes them to `gradle.lockfile`.

---

## R1. Foundation scope: this feature builds the shared infrastructure that 001 deferred

**Decision**: This feature builds the cross-cutting infrastructure that `specs/001-healthcheck-api/plan.md` (Complexity Tracking) explicitly deferred to "a future foundation feature":

- the shared `ErrorResponse` and `GlobalExceptionHandler` (Principle IX)
- correlation-ID handling and structured, sanitised logging (Principle XIV)
- the Principle XIII quality gates: `-Werror`, Checkstyle, SonarQube, OWASP dependency-check, a dependency-freshness check, JaCoCo, the unit/integration/functional/smoke suites, GitHub Actions CI, and CodeQL

**Rationale**:
- The spec requires the shared error shape (FR-025) and authentication (FR-017).
- The constitution gives business endpoints no exemption from correlation IDs (XIV) or from the quality gates (XIII).
- 001's deferrals were justified only because the healthcheck had no error path, no logging and no business request. None of that applies here.
- The feature request says repo-wide rules "should be enforced by /speckit.plan, /speckit.tasks, /speckit.implement".
- Deferring again would mean shipping a business endpoint whose error bodies and traceability break the constitution.

**Alternatives considered**:
- *A separate "foundation" feature first.* Rejected. It would make 002 depend on unplanned work, and the reference-data endpoints are the first real consumer that can shape and test that infrastructure.
- *Build only IX/XIV now and defer XIII again.* Rejected. XIII's deferral in 001 hinged on there being "one test class's worth" of coverage. This feature adds tests across all four suites.

---

## R2. Authentication: a custom servlet filter, not Spring Security

**Decision**: A single `BearerTokenAuthenticationFilter` (`OncePerRequestFilter`, `filters/`) protects every path under `/api/`, except `/api/v1/healthcheck`. That exemption is a code constant, not configuration.

**Configuration**: accepted tokens come from `jo.security.bearer-tokens`. `application.yml` sets it as `bearer-tokens: ${JO_SECURITY_BEARER_TOKENS:local-dev-token}`, so the environment variable `JO_SECURITY_BEARER_TOKENS` overrides the default with a comma-separated list (e.g. `a,b`). The placeholder is needed because Spring's relaxed binding would otherwise expect `JO_SECURITY_BEARERTOKENS` (dashes are removed), and `JO_SECURITY_BEARER_TOKENS` would silently bind to nothing (finding F1). An empty value (`JO_SECURITY_BEARER_TOKENS=`) resolves to an empty list, which fails the startup check below.

**Which path is checked**: the filter decides whether a request is protected using the path as the servlet container has **decoded and normalised** it: `request.getServletPath()` plus `request.getPathInfo()` when non-null (`DispatcherServlet` is mapped to `/`, so `getPathInfo()` is normally null). It never uses the raw `getRequestURI()`. Spring MVC matches routes against decoded path segments, so a raw-URI prefix check could be bypassed with an encoded path such as `/%61pi/v1/reference_data/...` (`%61` = `a`), which the raw check would treat as outside `/api/` while MVC still routed it to the controller. The container has already removed dot segments (`/./`) and path parameters (`;x=1`) from the servlet path, and Tomcat rejects encoded slashes (`%2F`) by default. `CorrelationIdFilter` uses the same path for its healthcheck exemption, so the two filters agree about which request is which. Both read it from one shared helper, `filters/RequestPaths.path(HttpServletRequest)`.

**Evaluation rules**:
- No `Authorization` header counts as **missing**.
- A scheme other than `Bearer` (case-insensitive), an empty token, or a token not in the list counts as **invalid**.
- Tokens are compared in constant time (`MessageDigest.isEqual` on UTF-8 bytes).
- Either failure returns `401` in the shared error shape with `error: "Unauthorized. Invalid or missing token."` (the E-Links wording) and a `WWW-Authenticate: Bearer` header (RFC 6750 §3).

**Startup validation**: the application refuses to start if the token list is empty or contains a blank entry. This is how "no setting, including an empty token set, may serve routes without a token" (FR-017; Clarification 2026-09-23) is enforced.

**Rationale**:
- Principle IX asks for simulated authentication that is centralised and configurable. Principle XII asks for no unnecessary abstraction.
- A 60-line filter meets both. Spring Security would bring a second filter chain, its own 401/403 bodies (which would need overriding to meet FR-025), and CSRF and session defaults that don't apply to a stateless mock.
- Protecting `/api/**` by default means later business endpoints are authenticated without anyone remembering to opt in (FR-020).
- Keeping the healthcheck exemption out of configuration means nobody can widen it by editing configuration (Clarification: authentication can't be turned off).
- `/v3/api-docs` and `/swagger-ui/**` sit outside `/api/` and stay unauthenticated. This is recorded as a Principle XV decision in plan.md.

**Alternatives considered**:
- *Spring Security resource server with opaque tokens.* Too heavy, and it doesn't produce the shared error body.
- *A configurable list of unauthenticated paths.* Rejected: it would work as an "auth off" switch.
- *Signed JWTs.* Rejected in the spec clarifications.

---

## R3. Where the checks run, and in what order

**Decision**: Checks run in this order. The first one that fails decides the response.

| # | Stage | Component | Failure |
|---|-------|-----------|---------|
| 1 | Correlation ID | `CorrelationIdFilter` (highest-precedence filter) | — (never fails) |
| 2 | Authentication | `BearerTokenAuthenticationFilter` | `401` |
| 3 | Route match | Spring MVC handler mapping (exact match, no trailing-slash match) | `404` (unmatched path, trailing slash), `405` (method) |
| 4 | Query parameters | `NoQueryParametersInterceptor` (`HandlerInterceptor`, registered for `/api/v1/reference_data/**`) | `400` |
| 5 | Attribute name | `ReferenceDataTypeRegistry.resolve(name)` | `400` |
| 6 | `reference_id` format | `ReferenceIdParser.parse(raw)` | `400` |
| 7 | Lookup | `ReferenceDataService` → repository | `404` |
| — | Anything unexpected | `GlobalExceptionHandler` catch-all, plus a JSON `/error` controller for errors raised outside Spring MVC | `500` |

**Rationale**:
- Authenticating before routing satisfies EC-009 and EC-013: unauthenticated callers can't probe which routes or types exist.
- Spring Framework 7 no longer treats a trailing slash as a match (the change began in 6.0), so EC-002 gets `404` with no extra code.
- An exception thrown from an interceptor's `preHandle` is still passed to `@RestControllerAdvice`, so the query-parameter check shares the one error path.

**Alternatives considered**:
- *Checking query parameters inside the controller.* Rejected: it puts a cross-cutting rule in the endpoint (Principle III), and every future reference-data route would need to repeat it.

---

## R4. Parsing `reference_id`

**Decision**: The `{reference_id}` path variable is bound as a `String` and passed to a shared `ReferenceIdParser` (`services/`).

- **Accepted**: values matching `^[0-9]+$`, parsed as `long`.
- **Rejected with `400`**: anything else, including `abc`, `12x`, `1.5`, `-1`, `+1`, ` 1`, `1e3`.
- **Leading zeros**: `007` is accepted and resolves to 7.
- **Overflow**: a digits-only value too large for `long` becomes "no such id", which is `404` (EC-003).

**Rationale**: Binding directly to `long` would accept `+1` and `-1`, and would turn overflow into a `400` type-mismatch error. That contradicts FR-015 and EC-003.

**Alternatives considered**:
- *`@Pattern` Bean Validation on the path variable.* It would give `400` for malformed ids, but not the overflow-to-`404` rule, and it would bring a second validation error path into `GlobalExceptionHandler`.

---

## R5. A generic reference-data mechanism driven by configuration

**Decision**: Reference-data types are declared in configuration, not in code:

```yaml
jo:
  reference-data:
    types:
      - name: appointment_titles          # canonical attribute name
        aliases: [appointment_title]      # deprecated aliases
        fixture: classpath:reference-data/appointment_titles.json
```

**Components**:
- **`ReferenceDataTypeRegistry`** (`services/`): the single point that resolves an attribute name or alias to a `ReferenceDataType` (FR-005). It is built once at startup from `ReferenceDataProperties`. Startup fails if any canonical name or alias is duplicated, blank, or used as both a name and an alias.
- **`FixtureReferenceDataRepository`** (`repository/`): loads each type's fixture at startup and checks the data rules (see data-model.md). It holds the records in an unmodifiable map ordered by `id`.
- **One controller**: `ReferenceDataController`, with two `@GetMapping`s parameterised by `{attribute_name}`. It contains no type-specific code.

**Adding a type later**: add one list entry and one fixture file. No Java changes (FR-008, SC-007). AC-019 is proven by an integration test that declares an extra `test_widgets` type through test properties.

**Rationale**:
- Principle VII asks for one mechanism with aliases resolved in one place; Principle X asks for configuration over hard-coding. A config-declared type list is the smallest design that meets both and lets a test register a type.
- The E-Links Swagger documents one shared `ReferenceDataResponse` schema for all eleven types, so one typed record shape (plus `name`, deviation D-1) matches the contract.
- Only `appointment_titles` is declared in the shipped `application.yml`, so no other type is served (FR-007).

**Alternatives considered**:
- *A Java `enum` of types.* Rejected: tests can't extend it (so AC-019 couldn't be proven), and each new type would be a code change.
- *One Spring bean ("provider") per type.* Rejected for now (Principle XII): there is no behaviour per type, only data. If a later type needs its own filtering, a provider interface can be added at that point.

---

## R6. The synthetic AppointmentTitle fixture

**Decision**: The fixture is one checked-in JSON file, `src/main/resources/reference-data/appointment_titles.json`. It is generated once during implementation from the 194 distinct `name` values in the reference extract (`eLinks_Pivotl_Production_all-data_2026-06-01_REF_AppointmentTitle.csv`). Only `name` is taken from the extract. Every other value is synthetic, following the rules in [data-model.md § Fixture generation rules](./data-model.md#fixture-generation-rules).

**Rationale**:
- Meets FR-021 (clarified: public title names only; synthetic ids, timestamps and dates) and FR-023 (gaps in the id sequence; a mix of null and non-null end dates).
- Checking in the generated file, rather than generating it at runtime, makes the ids stable across releases (FR-024) and deterministic (Principle V).
- The extract is not in this repository. It is in the sibling checkout `../ctam-jomockapi/joh-elinks-api/ReferenceData/`, even though CLAUDE.md says `joh-elinks-api/` is here. It is needed only once, to generate the file, so the build doesn't depend on it.

**Alternatives considered**:
- *Reusing the extract's own ids.* Rejected by clarification: they are production values.
- *Generating at startup with a seeded random generator.* Rejected: ids would change whenever the input list changes.

---

## R7. The shared error shape and framework-level errors

**Decision**:
- **Error body**: `record ErrorResponse(String error, Instant timestamp, String traceId)` in `exceptions/`.
- **`traceId`**: the request's correlation ID. `ErrorResponseFactory` reads MDC `correlationId` first, then falls back to the request attribute `CorrelationIds.ATTRIBUTE`, so errors rendered by `/error` after the filter chain has finished keep the ID (research R8). It is `null` only on the healthcheck path, which is exempt from correlation IDs.
- **`timestamp`**: taken from an injected `Clock` bean, so tests can fix it.
- **Who builds it**: one `ErrorResponseFactory`, used by `GlobalExceptionHandler`, the authentication filter and the JSON `/error` controller. That way, even errors raised outside Spring MVC (in filters, or failures after the response has started) use the same shape.
- **Framework exceptions** handled explicitly: `NoResourceFoundException` / `NoHandlerFoundException` → `404`, `HttpRequestMethodNotSupportedException` → `405`, `HttpMediaTypeNotAcceptableException` → `406`, and `Exception` → `500`.
- **The `406` body**: written straight to the servlet response as `application/json` by the shared writer. Normal content negotiation would itself reject a JSON body when the caller's `Accept` header excludes JSON. RFC 9110 §15.5.7 allows sending a non-acceptable representation.
- **Logging severity**: 4xx at `WARN`, 5xx at `ERROR`. Only the 5xx log includes the stack trace, and response bodies never do (FR-026).

**Error messages**:
- Fixed and generic, so they are deterministic and never echo caller input.
- They are listed in [contracts/reference-data-api.md](./contracts/reference-data-api.md#error-messages).

**Alternatives considered**:
- *RFC 9457 `ProblemDetail`.* Rejected: Principle IX specifies the `error` / `timestamp` / `traceId` shape.
- *Echoing the rejected attribute name in the message.* Rejected: it reflects caller input.

---

## R8. Correlation IDs and structured logging

**Decision**:

**`CorrelationIdFilter`** (`filters/`, highest precedence):
- Reads `X-Correlation-Id` from the request. The value is accepted only if it matches `^[A-Za-z0-9._-]{1,64}$`; otherwise a UUID is generated instead.
- Stores the ID under the MDC key `correlationId`, echoes it in the `X-Correlation-Id` response header, and clears the MDC in `finally`.
- Also stores the ID as the request attribute `CorrelationIds.ATTRIBUTE` (`uk.gov.hmcts.ctam.jo.correlationId`, in a small shared constants class `filters/CorrelationIds` with the header name and MDC key). An exception that escapes the filter chain is handled afterwards by the container's error dispatch to `/error`. By then this filter's `finally` has cleared the MDC, and `OncePerRequestFilter` doesn't run again on error dispatches. The request attribute survives the dispatch, so `/error` responses still carry the right `traceId` (finding C2). The container may also clear response headers before the `/error` dispatch, so `JsonErrorController` sets `X-Correlation-Id` again from the same attribute (finding C3).
- Skips `/api/v1/healthcheck`, the exemption for health-style endpoints (XIV). No other path is skipped.

**Logging**:
- Spring Boot's built-in structured logging over Logback: `logging.structured.format.console: logstash`. MDC entries appear in every log line automatically.
- Every caller-supplied value that is logged (attribute name, raw `reference_id`, inbound correlation header, request path) goes through `LogSanitiser.sanitise(...)`, which wraps OWASP Java Encoder `Encode.forJava`.
- Token values are never logged.

**Rationale**:
- Checking the inbound header's format blocks log and header injection through the one value that is echoed back.
- Using Spring Boot's built-in formatter avoids a hand-written Logback encoder.
- No shared HMCTS logging library is currently a dependency of this repo. Adopting one is not needed to meet XIV and can be looked at separately.

**Alternatives considered**:
- *A `logstash-logback-encoder` dependency.* Unnecessary, because Spring Boot already has built-in structured logging.

---

## R9. Response DTOs, entities and mapping

**Decision**:

**Response DTOs** (`domain/`), as Java records, matching the existing `HealthResponse` convention:
- `ReferenceDataApiResponse(List<ReferenceDataResponse> results)`
- `ReferenceDataResponse(id, name, createdAt, updatedAt, startDate, endDate)`, with `@JsonProperty` giving snake_case names and `@Schema` giving descriptions (XVIII).

**Internal model** (`entity/`):
- `ReferenceDataRecord`: Lombok `@Value @Builder @Jacksonized`, used for fixture deserialisation.
- `ReferenceDataType`: Lombok `@Value`.

**Mapping**: a MapStruct `ReferenceDataMapper` (`mappers/`, `componentModel = "spring"`) maps `ReferenceDataRecord` to `ReferenceDataResponse`, and a list to `ReferenceDataApiResponse`.

**Other Lombok use**: `@RequiredArgsConstructor` and `@Slf4j` on components.

**JSON output** (Jackson 3, `tools.jackson`, as used by Spring Boot 4.1 MVC):
- `LocalDate` → `"2024-01-01"`
- whole-second `Instant` → `"2024-01-15T09:00:00Z"`
- `null` `end_date` stays in the output as `null` (the default inclusion is ALWAYS)

The contract tests assert all three.

**Rationale**:
- Principle VIII: typed DTOs, internal model kept separate from the API DTOs, MapStruct for mapping, Lombok for boilerplate.
- Records already remove DTO boilerplate. Lombok goes where it earns its place: the builder for the fixture model, and constructor/logger injection.

**Alternatives considered**:
- *Serialising the entity directly.* Rejected: it ties the fixture file format to the API contract.
- *Lombok `@Value` classes for DTOs.* Rejected: inconsistent with `HealthResponse`.

---

## R10. OpenAPI documentation

**Decision**:
- springdoc annotations on `ReferenceDataController`: `@Operation`, and `@ApiResponse` for `200`, `400`, `401`, `404` (single-record route only), `406` and `500`. Every error response uses the `ErrorResponse` schema and has an example. `406` is listed because a `GET` with a non-JSON `Accept` header reaches it (spec EC-008), and Principle XVIII requires every status code an operation can return to be documented.
- A `bearerAuth` HTTP security scheme (`bearer`), applied to both operations.
- An `OpenApiCustomizer` bean (`config/OpenApiConfig`) fills in the `attribute_name` parameter's `enum` from `ReferenceDataTypeRegistry`, so the documented values always match the configured types. The alias is described as deprecated.

**Rationale**: Principle XVIII requires generated, reproducible documentation. Values injected from the registry can't drift from the configuration.

**Alternatives considered**:
- *Hard-coding `allowableValues` in annotations.* Rejected: it would drift from the configuration when a type is added.

---

## R11. Build, quality gates and test suites (Principle XIII)

**Decision**:

**Build plugins** (`build.gradle`):
- `uk.gov.hmcts.java` **0.12.70**: HMCTS's shared plugin, which wires Checkstyle and OWASP dependency-check together (the "single shared build-tool plugin"). OWASP suppressions go in `config/owasp/suppressions.xml`.
- `jacoco`, with `toolVersion` set to **0.8.15** (supports Java 25).
- `org.sonarqube` **7.5.0.8588**.
- `com.github.ben-manes.versions` **0.64.0**, providing the `dependencyUpdates` task.

**Compiler flags**: `options.compilerArgs += ['-Xlint:all,-processing', '-Werror']`. The `-processing` part stops Lombok and MapStruct annotation-processor notices from failing the build; every other warning is still an error.

**Dependencies**:
- `org.projectlombok:lombok` **1.18.48**
- `org.mapstruct:mapstruct` and `mapstruct-processor` **1.6.3** (latest stable; `1.7.0.Beta2` rejected as a pre-release)
- `org.projectlombok:lombok-mapstruct-binding` **0.2.0**
- `org.owasp.encoder:encoder` **1.4.0**

**Test suites** use Gradle's JVM Test Suite plugin (`testing { suites { … } }`):

| Suite | Source set | Contents |
|-------|------------|----------|
| `test` (unit) | `src/test/java` | Component unit tests plus `@WebMvcTest` controller tests |
| `integrationTest` | `src/integrationTest/java` | `@SpringBootTest` + MockMvc: filter order, framework 404/405/406 error shape, configuration overrides (AC-017), extension (AC-019), OpenAPI contract tests |
| `functionalTest` | `src/functionalTest/java` | `@SpringBootTest(RANDOM_PORT)` over real HTTP: every acceptance scenario in the spec, end to end |
| `smokeTest` | `src/smokeTest/java` | Minimal live round trips: healthcheck, plus one authenticated reference-data call |

- `check` depends on all four suites, `jacocoTestReport` and `dependencyCheckAggregate`.
- JaCoCo merges execution data from all suites into one report.

**Where the existing healthcheck tests go**:
- `HealthcheckControllerTest` stays in `test`.
- `HealthcheckOpenApiContractTest` moves to `integrationTest`.
- `HealthcheckSmokeTest` moves to `smokeTest`.

**CI**:
- `.github/workflows/ci.yml` runs `./gradlew check` on pull requests and pushes to `main`. Sonar runs as `./gradlew sonar` only when `SONAR_TOKEN` is set. OWASP uses the `NVD_API_KEY` secret. `dependencyUpdates` output is published as a job artefact.
- `.github/workflows/codeql.yml` (Java; weekly schedule plus PR and push) runs separately from the main build.

**Risks, checked in Setup**:
1. The HMCTS plugin may not yet support Gradle 9.7.1 or Java 25. **Fallback**: apply the core `checkstyle` plugin (HMCTS rule set checked in at `config/checkstyle/checkstyle.xml`) plus `org.owasp.dependencycheck` **13.0.0** directly. Either route satisfies XIII.
2. CodeQL's Java extractor may not yet support Java 25 bytecode or source. **Fallback**: a manual build mode that compiles with `./gradlew compileJava`. If CodeQL can't analyse Java 25 at all, record it in plan.md Complexity Tracking with the upstream issue as the way back to compliance.
3. `dependencyCheckAggregate` needs NVD data and is slow when run locally. Local `./gradlew check` accepts `-PskipOwasp`. CI never skips it.

**Rationale**: Each item maps directly to a Principle XIII clause, and all are standard Gradle and HMCTS tooling.

**Alternatives considered**:
- *Hand-rolled `sourceSets` plus `Test` tasks for the suites.* The JVM Test Suite plugin is the Gradle-native equivalent and needs less configuration.

---

## R12. How `@WebMvcTest` slices interact with the new filters

**Decision**:
- The filters and interceptor are `@Component`s whose configuration-properties beans are enabled in `config/`.
- `@WebMvcTest` slices automatically pick up `Filter`, `HandlerInterceptor` and `WebMvcConfigurer` beans. Each controller slice test therefore `@Import`s the shared web configuration and supplies test properties (e.g. `jo.security.bearer-tokens=test-token`).
- `HealthcheckControllerTest` gains the same import. Its behaviour doesn't change, because the healthcheck is exempt from both filters.

**Rationale**: Without these imports, adding the filters would break the existing healthcheck slice test when the context starts.

---

## R13. Performance

**Decision**: No caching layer. The data is 194 records held in memory, so each request is a map lookup or a list copy.

**Target**: p95 under 100 ms for both routes on a developer machine. The smoke suite enforces a coarse p95 < 1 s by default, so shared CI runners don't produce flaky failures. The strict 100 ms bound runs only with `-Dperf.strict=true` (tasks T038, T067).

**Rationale**: The collection response is roughly 30 KB of JSON, so there is no scale concern.

---

## Resolved: no outstanding `NEEDS CLARIFICATION` items

Every Technical Context unknown is resolved above. The spec's three deferred items are settled here:

| Deferred item | Settled in |
|---------------|------------|
| Fixture location and configuration | R5, R6 |
| Error message wording | R7 and the contract |
| `WWW-Authenticate` header | R2: included |
