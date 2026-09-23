---

description: "Task list for 002-reference-data-api"
---

# Tasks: Reference Data API — Appointment Titles

**Input**: Design documents from `/specs/002-reference-data-api/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/reference-data-api.md](./contracts/reference-data-api.md), [quickstart.md](./quickstart.md)

**Tests**: **Required.** Spec SC-001 says every acceptance scenario (AC-001 to AC-020) must pass as an automated test, and constitution Principle XI requires unit, controller/API and contract tests. Within each story, write the tests first and confirm they fail before implementing.

**Organization**: Setup and Foundational build the shared infrastructure that 001 deferred (research R1). The story phases follow spec priority: P1 (US1, US2, US5), then P2 (US3, US4), then P3 (US6).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies on incomplete tasks)
- **[Story]**: Which user story this task belongs to (US1–US6, matching spec.md)

## Path Conventions

This is a single Gradle project. Paths are from the repository root.

| Short form | Full path |
|------------|-----------|
| `main/` | `src/main/java/uk/gov/hmcts/ctam/jo/` |
| `unit/` | `src/test/java/uk/gov/hmcts/ctam/jo/` |
| `it/` | `src/integrationTest/java/uk/gov/hmcts/ctam/jo/` |
| `ft/` | `src/functionalTest/java/uk/gov/hmcts/ctam/jo/` |
| `smoke/` | `src/smokeTest/java/uk/gov/hmcts/ctam/jo/` |

**Shared test conventions**:
- The test token is `test-token` (`jo.security.bearer-tokens=test-token` in each suite's `application-test.yml` or `@TestPropertySource`).
- Expected error messages are the exact strings in [contracts/reference-data-api.md § Error messages](./contracts/reference-data-api.md#error-messages).
- Known fixture points are in [data-model.md § Known reference points](./data-model.md#fixture-generation-rules): id 10 = "Acting Senior Coroner", id 70 = "Area Coroner" (ended 2025-03-31), id 1940 = the last record; id 15 and id 999999 are unknown.

---

## Phase 1: Setup (Build, Quality Gates, CI)

**Purpose**: Deliver Principle XIII's build gates and the four test suites (research R11) before any feature code lands, so every later task is built under `-Werror`, Checkstyle and coverage.

- [ ] T001 Update `build.gradle` dependencies (research R9, R11):
  - `compileOnly` and `annotationProcessor` for `org.projectlombok:lombok:1.18.48`, and the same for `testCompileOnly`/`testAnnotationProcessor`
  - `implementation 'org.mapstruct:mapstruct:1.6.3'`
  - `annotationProcessor 'org.mapstruct:mapstruct-processor:1.6.3'`
  - `annotationProcessor 'org.projectlombok:lombok-mapstruct-binding:0.2.0'`
  - `implementation 'org.owasp.encoder:encoder:1.4.0'`

  Add `tasks.withType(JavaCompile).configureEach { options.compilerArgs += ['-Xlint:all,-processing', '-Werror'] }`, and add `lombok.config` at the repo root with `lombok.addLombokGeneratedAnnotation = true` so JaCoCo leaves generated code out.
- [ ] T002 Set up the test suites in `build.gradle` with the JVM Test Suite plugin (`testing { suites { … } }`, research R11):
  - Keep `test` as the unit suite.
  - Add `integrationTest`, `functionalTest` and `smokeTest` (`JvmTestSuite`, `useJUnitJupiter()`), each depending on `project()`, `spring-boot-starter-test`, `spring-boot-starter-webmvc-test`, `spring-boot-http-client` and `spring-boot-restclient`, plus Lombok for tests.
  - Make `check` depend on all four suites. Order them `integrationTest` after `test`, `functionalTest` after `integrationTest`, and `smokeTest` after `functionalTest`.
- [ ] T003 Move the existing `@SpringBootTest` healthcheck tests into their suites, changing each package declaration to `uk.gov.hmcts.ctam.jo` and keeping the test bodies as they are:
  - `src/test/java/uk/gov/hmcts/ctam/jo/controllers/HealthcheckOpenApiContractTest.java` → `it/HealthcheckOpenApiContractTest.java`
  - `src/test/java/uk/gov/hmcts/ctam/jo/controllers/HealthcheckSmokeTest.java` → `smoke/HealthcheckSmokeTest.java`

  `HealthcheckControllerTest` stays in `unit/controllers/`. Run `./gradlew test integrationTest smokeTest` and confirm all pass.
- [ ] T004 Add JaCoCo to `build.gradle`: the `jacoco` plugin with `toolVersion = '0.8.15'`, and a `jacocoTestReport` that merges `executionData` from all four suites, produces XML and HTML, and runs from `check`.
- [ ] T005 Add Checkstyle and OWASP dependency-check to `build.gradle` (research R11, risk 1):
  - Apply `id 'uk.gov.hmcts.java' version '0.12.70'` and run `./gradlew checkstyleMain dependencyCheckAggregate --dry-run`.
  - **If the plugin fails** under Gradle 9.7.1 / Java 25, remove it and instead apply the core `checkstyle` plugin (HMCTS rule set in `config/checkstyle/checkstyle.xml`) plus `id 'org.owasp.dependencycheck' version '13.0.0'`.
  - Either way, configure dependency-check to use `config/owasp/suppressions.xml`, fail on CVSS ≥ 7, and read `NVD_API_KEY` from the environment.
  - Add a `-PskipOwasp` project property that takes `dependencyCheckAggregate` out of `check` for local runs.
  - **Missing API key (finding E2)**: when the environment variable `CI` is `true`, `NVD_API_KEY` is unset or blank, and `-PskipOwasp` isn't given, `dependencyCheckAggregate` must fail at once. The failure message should say "NVD_API_KEY is not set; OWASP dependency-check cannot run in CI". The scan must never run keyless, which is slow and rate-limited, and must never be skipped silently, because Principle XIII forbids skipped gates. Outside CI, a missing key logs a `WARN` and the scan runs keyless.
  - Create `config/owasp/suppressions.xml` as a valid, empty `<suppressions>` document with a header comment requiring a documented reason for each entry.
  - Record which route was taken in a comment in `build.gradle`.
- [ ] T006 Add Sonar and dependency freshness to `build.gradle`:
  - `id 'org.sonarqube' version '7.5.0.8588'`, with `sonar { properties { property 'sonar.projectKey', 'hmcts_ctam-jomockapi2'; property 'sonar.organization', 'hmcts'; property 'sonar.host.url', 'https://sonarcloud.io'; property 'sonar.coverage.jacoco.xmlReportPaths', <jacoco xml path> } }`. The key and organisation were confirmed on 2026-09-23 against the existing public SonarQube Cloud project (`https://sonarcloud.io/api/components/show?component=hmcts_ctam-jomockapi2`, last analysed that day). That project already analyses PRs even though the repo has no CI or Sonar config, which suggests SonarQube Cloud **Automatic Analysis** is on. Automatic Analysis and a CI `./gradlew sonar` run can't both be enabled, and Automatic Analysis doesn't import JaCoCo coverage. So a SonarQube Cloud project admin must switch Automatic Analysis off (Administration → Analysis Method) and add the `SONAR_TOKEN` repository secret before T008's sonar step is enabled. Until then, T008's `if: env.SONAR_TOKEN != ''` guard keeps CI green, and Automatic Analysis carries on as it is. This is a documented Principle XIII deferral: see plan.md § Complexity Tracking, "Sonar step in CI stays inactive". Record which state applies in the PR description (finding B2). Before the PR merges, raise a GitHub issue asking a SonarQube Cloud admin to switch Automatic Analysis off and add `SONAR_TOKEN`, and link it from the PR description. If the secret already exists when this task runs, leave out the guard and treat the deferral as closed.
  - `id 'com.github.ben-manes.versions' version '0.64.0'`, with `dependencyUpdates { rejectVersionIf { isNonStable(it.candidate.version) } }`, where the non-stable check rejects `alpha|beta|rc|cr|m|preview|snapshot` qualifiers
- [ ] T007 Regenerate the lock file for every configuration, including the new suites: `./gradlew dependencies --write-locks` and `./gradlew integrationTestCompileClasspath functionalTestRuntimeClasspath smokeTestRuntimeClasspath --write-locks` (or `resolveAndLockAll`). Then run `./gradlew check -PskipOwasp` and confirm it is green on the existing healthcheck code. Fix any Checkstyle findings in `HealthcheckController.java`, `HealthResponse.java`, `Application.java` and the moved tests.
- [ ] T008 [P] Create `.github/workflows/ci.yml` (research R11):
  - Triggers: `pull_request`, and `push` to `main`.
  - Job: `ubuntu-latest`, `actions/setup-java` with Temurin 25, `gradle/actions/setup-gradle`.
  - Steps: `./gradlew check` with `NVD_API_KEY: ${{ secrets.NVD_API_KEY }}`. Never pass `-PskipOwasp` in CI. If the secret is missing, the T005 guard fails the build loudly (finding E2). PRs from forks don't receive secrets, so a maintainer must re-run them from a branch in this repository before merge; state this in a comment in the workflow; then `./gradlew sonar` only `if: env.SONAR_TOKEN != ''`, with a comment above the step pointing to plan.md § Complexity Tracking and saying the guard must be removed once `SONAR_TOKEN` is configured; then `./gradlew dependencyUpdates`, uploading `build/dependencyUpdates/` and the JaCoCo and test reports as artefacts.
- [ ] T009 [P] Create `.github/workflows/codeql.yml` as a separate workflow, not part of CI:
  - Triggers: weekly `schedule` (cron), `pull_request`, and `push` to `main`.
  - Steps: `github/codeql-action/init` with `languages: java-kotlin` and `build-mode: manual`, a Temurin 25 setup, `./gradlew compileJava -x test`, then `analyze`.

  Check CodeQL's documented Java version support (research R11, risk 2). If Java 25 is not yet supported, set `continue-on-error: true` on the analyze step and add the conditional row to `specs/002-reference-data-api/plan.md` § Complexity Tracking, naming the way back to compliance.

**Checkpoint**: `./gradlew check -PskipOwasp` passes. All four suites run. JaCoCo and Checkstyle reports are produced.

---

## Phase 2: Foundational (Shared Infrastructure)

**Purpose**: The error shape, correlation IDs, logging, authentication, the reference-data type registry, the fixture repository and the mapper. Every story needs these.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

### Error handling, correlation IDs and logging

- [ ] T010 [P] Create `main/exceptions/ErrorResponse.java`: `public record ErrorResponse(String error, Instant timestamp, String traceId)`, with `@Schema` descriptions on each component matching the contract's `ErrorResponse` table (research R7).
- [ ] T011 [P] Create the four exception classes in `main/exceptions/`, each a `RuntimeException` with the contract's fixed message as a constant:
  - `UnsupportedReferenceDataTypeException`
  - `InvalidReferenceIdException`
  - `ReferenceDataNotFoundException`
  - `UnsupportedQueryParameterException`

  Constructors may take a *diagnostic* detail (the sanitised raw value) for logs only. `getMessage()` must return the fixed contract message.
- [ ] T012 [P] Create `main/util/LogSanitiser.java`: a final utility class with `static String sanitise(String value)`, which returns `Encode.forJava(value)` and returns the string `"null"` for `null`. Write `unit/util/LogSanitiserTest.java` covering CR/LF, tab, quotes, non-ASCII and `null` (research R8). Also create `main/filters/CorrelationIds.java`: a final constants class with `HEADER = "X-Correlation-Id"`, `MDC_KEY = "correlationId"` and `ATTRIBUTE = "uk.gov.hmcts.ctam.jo.correlationId"`, used by T014, T016 and T017, so no task depends on a class created later (finding F2).
- [ ] T013 Create `main/config/WebConfig.java` (`@Configuration`): a `Clock` bean (`Clock.systemUTC()`). Also add `@ConfigurationPropertiesScan` to `main/Application.java`, so `SecurityProperties` (T019) and `ReferenceDataProperties` (T022) are registered automatically when they land. Nothing references them before they exist (finding F2). Leave a clearly marked `addInterceptors` override stub for T046 (research R3, R7).
- [ ] T014 Create `main/exceptions/ErrorResponseFactory.java` (`@Component`, `@RequiredArgsConstructor`, with an injected `Clock` and Spring's `JsonMapper`, Jackson 3):
  - `ErrorResponse create(HttpServletRequest request, String message)`: `traceId` is `MDC.get(CorrelationIds.MDC_KEY)`, or, if that is null, `request.getAttribute(CorrelationIds.ATTRIBUTE)`. It is `null` only when neither is set (the exempt healthcheck path).
  - `void write(HttpServletRequest request, HttpServletResponse response, HttpStatus status, String message)`: sets the status and `Content-Type: application/json`, then serialises `create(request, message)` directly. This is used by filters and for `406`.

  Write `unit/exceptions/ErrorResponseFactoryTest.java` with a fixed `Clock`, covering: MDC set; MDC unset but the request attribute set (the attribute is used); both unset (`null`) (research R7).
- [ ] T015 Create `main/exceptions/GlobalExceptionHandler.java`, the single `@RestControllerAdvice`, using `@Slf4j` and `ErrorResponseFactory`. It maps:

  | Exception | Status |
  |-----------|--------|
  | `UnsupportedReferenceDataTypeException`, `InvalidReferenceIdException`, `UnsupportedQueryParameterException` | `400` |
  | `ReferenceDataNotFoundException` | `404`, "Reference data record not found." |
  | `NoResourceFoundException`, `NoHandlerFoundException` | `404`, "Resource not found." |
  | `HttpRequestMethodNotSupportedException` | `405` |
  | `HttpMediaTypeNotAcceptableException` | `406`, written with `ErrorResponseFactory.write`, returning `void` (research R7) |
  | `Exception` | `500`, "Internal server error." |

  Log 4xx at `WARN` without a stack trace, and 5xx at `ERROR` with the stack trace. Pass every logged caller value through `LogSanitiser`. Write `unit/exceptions/GlobalExceptionHandlerTest.java` asserting the status, the exact message and that no internals appear in the body for each mapping.
- [ ] T016 Create `main/exceptions/JsonErrorController.java`, implementing Spring Boot's `ErrorController` and mapped to `/error`. It reads `RequestDispatcher.ERROR_STATUS_CODE` and returns `ErrorResponse` with the contract message for that status (`500` if it is unknown), and never exposes the exception or stack trace. It also sets the `X-Correlation-Id` response header from `request.getAttribute(CorrelationIds.ATTRIBUTE)` whenever that attribute is present, because the container may clear headers set earlier in the filter chain before it calls `/error` (finding C3). It uses `@Slf4j` and follows Principle IX's severity rule. A 5xx is logged at `ERROR`, with the `RequestDispatcher.ERROR_EXCEPTION` stack trace when one is present. A 4xx is logged at `WARN` without a stack trace. Each log line carries the sanitised `RequestDispatcher.ERROR_REQUEST_URI` (through `LogSanitiser`), and the correlation ID from the request attribute is put back into MDC for the duration of the log call (finding U1). Add `server.error.include-stacktrace: never` and `server.error.include-message: never` to `src/main/resources/application.yml`.
- [ ] T017 First create `main/filters/RequestPaths.java`: a final utility class with `static String path(HttpServletRequest request)` returning `request.getServletPath()` plus `request.getPathInfo()` when non-null. This is the path **as decoded and normalised by the container**. Filters must never use `getRequestURI()` for access or exemption decisions (research R2, finding C1). Write `unit/filters/RequestPathsTest.java` covering: servlet path only, servlet path plus path info, and path info null.

  Then create `main/filters/CorrelationIdFilter.java`:
  - A `OncePerRequestFilter` `@Component` at `@Order(Ordered.HIGHEST_PRECEDENCE)`.
  - Uses the inbound `CorrelationIds.HEADER` (`X-Correlation-Id`) if it matches `^[A-Za-z0-9._-]{1,64}$`; otherwise generates `UUID.randomUUID().toString()`.
  - Puts the ID in MDC under `CorrelationIds.MDC_KEY`, **and** in the request attribute `CorrelationIds.ATTRIBUTE` (both from T012). Sets the `CorrelationIds.HEADER` response header before calling the chain, and removes the MDC entry in `finally`. The request attribute stays, so the ID is still available on the `/error` dispatch (research R8, finding C2).
  - `shouldNotFilter` returns true only when `RequestPaths.path(request)` equals `/api/v1/healthcheck` (research R2, R8).

  Write `unit/filters/CorrelationIdFilterTest.java` covering: a valid header echoed, invalid headers replaced (CRLF, 65 characters, spaces), a missing header generated, the MDC cleared afterwards while the request attribute is still set, and the healthcheck skipped.
- [ ] T018 Configure structured logging in `src/main/resources/application.yml`: `logging.structured.format.console: logstash` (research R8). Confirm with `./gradlew bootRun` that one request produces a JSON log line containing `correlationId`.

### Authentication

- [ ] T019 Create `main/config/SecurityProperties.java`:
  - `@ConfigurationProperties("jo.security")` and `@Validated`, as a record with `List<String> bearerTokens`.
  - Validate in the compact constructor: the list must be non-null and non-empty with no blank entries. On failure, throw `IllegalArgumentException("jo.security.bearer-tokens must contain at least one non-blank token")` so startup fails (research R2).

  Add to `src/main/resources/application.yml`:
  ```yaml
  jo:
    security:
      # NON-SECRET default for local/dev/Postman. Override with JO_SECURITY_BEARER_TOKENS (comma-separated).
      # The explicit placeholder is required: relaxed binding would otherwise expect JO_SECURITY_BEARERTOKENS.
      bearer-tokens: ${JO_SECURITY_BEARER_TOKENS:local-dev-token}
  ```
- [ ] T020 Create `main/filters/BearerTokenAuthenticationFilter.java`:
  - A `OncePerRequestFilter` `@Component` at `@Order(Ordered.HIGHEST_PRECEDENCE + 10)`, with `@RequiredArgsConstructor` and `@Slf4j`.
  - `shouldNotFilter` evaluates `RequestPaths.path(request)`, never the raw URI. It returns true for paths that don't start with `/api/`, and for exactly `/api/v1/healthcheck`. That exemption is a `private static final` constant, not configuration.
  - Parse `Authorization`. The request passes only if the scheme is `Bearer` (case-insensitive), followed by one space and a non-empty token that matches one configured token under `MessageDigest.isEqual` on UTF-8 bytes.
  - Otherwise, set `WWW-Authenticate: Bearer` and call `ErrorResponseFactory.write(request, response, UNAUTHORIZED, "Unauthorized. Invalid or missing token.")`.
  - Log a `WARN` with the sanitised path and the reason category ("missing" or "invalid"). Never log the header or token value (research R2, R8).

  Write `unit/filters/BearerTokenAuthenticationFilterTest.java` covering: a valid token; lowercase `bearer`; a missing header; `Basic abc`; `Bearer`; `Bearer ` (empty); `Bearer wrong`; a token with a trailing space; the healthcheck exempt; `/v3/api-docs` not filtered; a non-`/api/` path not filtered; a request whose raw `requestURI` is `/%61pi/v1/reference_data/appointment_titles` but whose `servletPath` is `/api/v1/reference_data/appointment_titles` → filtered (`401` without a token).

### Reference-data core

- [ ] T021 [P] Create `main/entity/ReferenceDataType.java` (Lombok `@Value`: `String name`, `Set<String> aliases`, `String fixture`) and `main/entity/ReferenceDataRecord.java` (Lombok `@Value @Builder @Jacksonized`, with fields `id`, `name`, `createdAt`, `updatedAt`, `startDate`, `endDate` as in data-model.md §2; `@JsonProperty` gives the snake_case names; `@JsonIgnoreProperties(ignoreUnknown = false)`).
- [ ] T022 [P] Create `main/config/ReferenceDataProperties.java`: `@ConfigurationProperties("jo.reference-data")`, as a record with `List<TypeProperties> types`, where `record TypeProperties(String name, List<String> aliases, String fixture)` and `aliases` defaults to an empty list (data-model.md §1). `types` also defaults to an empty list when `jo.reference-data.types` is unset (normalise `null` in the compact constructor). The Phase 2 checkpoint runs full-context tests before T036 declares any type (finding C3).
- [ ] T023 Create `main/services/ReferenceDataTypeRegistry.java` (`@Component`), built from `ReferenceDataProperties` at construction:
  - Check every name and alias against `^[a-z][a-z_]*$`.
  - Reject duplicates across all names and aliases, and any type that lists its own name as an alias. On any failure, throw `IllegalStateException` so startup fails.
  - `ReferenceDataType resolve(String attributeName)`: an exact, case-sensitive lookup that throws `UnsupportedReferenceDataTypeException` when nothing matches.
  - `List<String> supportedAttributeNames()`: canonical names first, then aliases, in configuration order (used by OpenAPI in T034).
  - Class Javadoc: "the ONLY place reference-data names and deprecated aliases are resolved (spec FR-005)".

  - Zero declared types is valid: the registry builds, `resolve` throws `UnsupportedReferenceDataTypeException` for every name, and `supportedAttributeNames()` is empty (finding C3).

  Write `unit/services/ReferenceDataTypeRegistryTest.java` covering: canonical name, alias, unknown, case variants (`Appointment_Titles`), zero types, and each startup-validation failure.
- [ ] T024 Create `main/repository/ReferenceDataRepository.java` (interface: `List<ReferenceDataRecord> findAll(ReferenceDataType type)` and `Optional<ReferenceDataRecord> findById(ReferenceDataType type, long id)`) and `main/repository/FixtureReferenceDataRepository.java` (`@Repository`):
  - At construction, load each registry type's `fixture` through `ResourceLoader` and Jackson 3 (`FAIL_ON_UNKNOWN_PROPERTIES` on).
  - Check the data-model.md §2 rules: id > 0 and unique; name non-blank, trimmed and unique; required fields present; `createdAt ≤ updatedAt`; `endDate ≥ startDate`.
  - On any failure, throw `IllegalStateException("Invalid reference-data fixture <fixture>: record id <id>: <rule>")`.
  - Store the records per type in an unmodifiable map sorted by id (a `TreeMap` copy). `findAll` returns an unmodifiable list in ascending id order.

  - With zero registry types, construction succeeds and loads nothing (finding C3).

  Write `unit/repository/FixtureReferenceDataRepositoryTest.java` with small fixtures under `src/test/resources/reference-data/`: valid unsorted input returned sorted; an empty array giving an empty list; zero registry types; and one failing fixture per rule, including an unknown property.
- [ ] T025 [P] First create `main/domain/ReferenceDataResponse.java` and `main/domain/ReferenceDataApiResponse.java` as records, using the field names, `@JsonProperty` snake_case names, `@JsonPropertyOrder({"id","name","created_at","updated_at","start_date","end_date"})` and `@JsonInclude(JsonInclude.Include.ALWAYS)` on `endDate` from data-model.md §3. (T031 adds their OpenAPI `@Schema` docs.) Then create `main/mappers/ReferenceDataMapper.java`: a MapStruct `@Mapper(componentModel = "spring")` with `ReferenceDataResponse toResponse(ReferenceDataRecord record)` and `List<ReferenceDataResponse> toResponses(List<ReferenceDataRecord> records)`. Write `unit/mappers/ReferenceDataMapperTest.java` using `Mappers.getMapper`, covering a null `endDate` and order being kept.

### Foundation checks

- [ ] T026 Update `unit/controllers/HealthcheckControllerTest.java` to `@Import({WebConfig.class, ErrorResponseFactory.class, CorrelationIdFilter.class, BearerTokenAuthenticationFilter.class})` with `@TestPropertySource(properties = "jo.security.bearer-tokens=test-token")` and `@EnableConfigurationProperties(SecurityProperties.class)`, so the slice has the properties bean whether or not it picks up `@ConfigurationPropertiesScan` from `Application` (the explicit registration is harmless if both apply). Confirm all its existing tests still pass without a token (research R12; spec AC-018).
- [ ] T027 Write `it/ErrorShapeIntegrationTest.java` (`@SpringBootTest` + `@AutoConfigureMockMvc`, token `test-token`). Both test classes in this task set the token with `@TestPropertySource(properties = "jo.security.bearer-tokens=test-token")`, because the suites' `application-test.yml` files arrive only in T036. T036 may switch them to `@ActiveProfiles("test")` afterwards (finding C4). Assert the shared `ErrorResponse` shape (exactly the keys `error`, `timestamp`, `traceId`), the exact message and the `X-Correlation-Id` header for:
  - `GET /api/v1/does-not-exist` with a token → `404` "Resource not found."
  - `POST /api/v1/healthcheck` → `405` with `traceId` null (healthcheck exempt from correlation)
  - `GET /api/v1/does-not-exist` without a token → `401` (authentication runs before routing; spec EC-013)
  - a supplied `X-Correlation-Id: it-123` echoed in the header and in `traceId`
  - `GET /api/v1/healthcheck` → `200` with no `X-Correlation-Id` header

  Also write `it/ErrorDispatchIntegrationTest.java` (**finding C2**), using `@SpringBootTest(webEnvironment = RANDOM_PORT)` because MockMvc doesn't perform the container's `/error` dispatch. Register a test-only filter (a `@TestConfiguration` `FilterRegistrationBean`, ordered after `BearerTokenAuthenticationFilter` and mapped to `/api/v1/test-escape`) that throws `IllegalStateException("escaped /secret")`. Call that path over real HTTP with a token and `X-Correlation-Id: esc-123`. Expect `500` "Internal server error." from `JsonErrorController`, with `traceId` `esc-123`, the response header `X-Correlation-Id: esc-123`, and a body containing neither `escaped` nor `IllegalStateException`.

**Checkpoint**: `./gradlew check -PskipOwasp` passes. The foundation is ready, and user stories can start.

---

## Phase 3: User Story 1 — Integrator Retrieves All Appointment Titles (Priority: P1) 🎯 MVP

**Goal**: `GET /api/v1/reference_data/appointment_titles` returns all 194 synthetic records as `{ "results": [...] }`.

**Independent Test**: With `Authorization: Bearer <token>`, the collection returns `200` with 194 records in ascending `id` order, identical on repeated calls (AC-001, AC-012, AC-015).

### Tests for User Story 1 ⚠️ (write first; confirm they fail)

- [ ] T028 [P] [US1] Write `unit/controllers/ReferenceDataControllerTest.java`:
  - `@WebMvcTest(ReferenceDataController.class)`, importing the same beans as T026, with `@MockitoBean ReferenceDataService`.
  - Collection cases: `200`, `application/json`, `$.results` as an array, the snake_case field names, `end_date` present as `null`, and the service called with the path's attribute name.
- [ ] T029 [P] [US1] Write `ft/ReferenceDataFunctionalTest.java`:
  - `@SpringBootTest(webEnvironment = RANDOM_PORT)` with `TestRestTemplate`, header `Authorization: Bearer test-token`, organised with a JUnit `@Nested` class per user story.
  - `@Nested CollectionTests` (US1):
    - **AC-001**: `200`, `Content-Type` `application/json`, `results.length == 194`, ids strictly ascending, and ids 10, 70 and 1940 with the data-model.md names and end dates.
    - **AC-012**: `GET /api/v5/reference_data/appointment_titles` and `GET /reference_data/appointment_titles` → `404`.
    - **AC-015 / SC-003**: 100 sequential calls produce identical bodies, and the body equals the checked-in golden file `src/functionalTest/resources/golden/appointment_titles.json` byte for byte, which proves the output is the same across restarts.
    - **Principle VIII**: no other fields; every record has exactly 6 keys.
- [ ] T030 [P] [US1] Write `it/ReferenceDataOpenApiContractTest.java` (`@SpringBootTest(RANDOM_PORT)`, fetching `/v3/api-docs`, in the style of `HealthcheckOpenApiContractTest`). For the collection path, assert:
  - `operationId` `getReferenceData` and tag `Reference Data`
  - the `attribute_name` path parameter's `enum` equals `["appointment_titles"]`, extended in T054
  - responses `200` (with `ReferenceDataApiResponse`), and `400`, `401`, `406` and `500` (with `ErrorResponse`), with no other codes
  - the `bearerAuth` security requirement, and `components.securitySchemes.bearerAuth` of type `http`, scheme `bearer`
  - `ReferenceDataResponse` properties equal the contract's six fields, with `end_date` nullable and date formats as in the contract
  - **AC-012 (finding E3)**: every key under `paths` that contains `reference_data` starts with `/api/v1/reference_data/`, and there are exactly two such paths: the collection and the single-record route
  - **Headers (finding D2)**: the collection operation has an `X-Correlation-Id` parameter with `in: header`, `required: false` and the T034 `pattern`. Every response declares an `X-Correlation-Id` response header, and the `401` response also declares `WWW-Authenticate`.

### Implementation for User Story 1

- [ ] T031 [P] [US1] Add the OpenAPI documentation to the two DTO records created in T025 (`main/domain/ReferenceDataResponse.java`, `main/domain/ReferenceDataApiResponse.java`): `@Schema` descriptions and examples on every component, taken from the contract's schema tables, with `@Schema(nullable = true)` on `endDate` and `format = "date"` / `"date-time"` where appropriate (Principle XVIII).
- [ ] T032 [US1] Generate `src/main/resources/reference-data/appointment_titles.json` once, following [data-model.md § Fixture generation rules](./data-model.md#fixture-generation-rules):
  - Read only the `name` column of `../ctam-jomockapi/joh-elinks-api/ReferenceData/eLinks_Pivotl_Production_all-data_2026-06-01_REF_AppointmentTitle.csv`.
  - Sort by code point. Set `id = 10 × position`. Records at positions where `position % 7 == 0` get `end_date` 2025-03-31 and `updated_at` 2025-04-01T08:00:00Z; the rest get `null` and 2024-06-03T10:30:00Z. Every record gets `created_at` 2024-01-15T09:00:00Z and `start_date` 2024-01-01.
  - Write pretty-printed JSON with 2-space indentation and a trailing newline.

  Use a throwaway script and don't commit it. Check: 194 records, 27 with an end date, id 10 = "Acting Senior Coroner", id 70 = "Area Coroner", id 1940 = "Vice-President, Employment Tribunal (Scotland)". Then create the golden file `src/functionalTest/resources/golden/appointment_titles.json` by capturing the actual compact response body once T035 is working. It is **not** the same as the fixture.
- [ ] T033 [US1] Create `main/services/ReferenceDataService.java` (`@Service`, `@RequiredArgsConstructor`), which uses the registry, repository and mapper: `ReferenceDataApiResponse getAll(String attributeName)` resolves the type, calls `findAll` and maps the result. Write `unit/services/ReferenceDataServiceTest.java` with mocks, covering: the correct type passed to the repository, the order kept, and an empty list giving `results: []`.
- [ ] T034 [US1] Create `main/config/OpenApiConfig.java` (research R10):
  - An `OpenAPI` bean with info title "JO Mock API" and `components.securitySchemes.bearerAuth` (`type: http`, `scheme: bearer`).
  - An `OpenApiCustomizer` bean that, for every operation under `/api/v1/reference_data/`, sets the `attribute_name` path parameter's schema `enum` to `registry.supportedAttributeNames()`, and builds its description entirely from the registry: `"Can be one of: " + <canonical names joined by ", ">`, plus `". Also supports deprecated values: " + <aliases joined by ", ">` when any aliases exist. No type name is written literally in the code (T061; finding F5).
  - The same customizer also documents the correlation and authentication headers on every `/api/v1/reference_data/` operation, so Principle XVIII's "request headers" requirement is met in one place (Principle III; finding D2):
    - an optional **request** header parameter `X-Correlation-Id` (`in: header`, `required: false`, schema `string` with `pattern` `^[A-Za-z0-9._-]{1,64}$`), described as "Optional correlation ID. Used if it matches the pattern; otherwise a UUID is generated. Echoed in the `X-Correlation-Id` response header and in `traceId` on errors."
    - an `X-Correlation-Id` **response** header (schema `string`) on every response of those operations
    - a `WWW-Authenticate` response header (schema `string`, example `Bearer`) on each `401` response

    Build the header objects once and reuse them. Don't add them per controller method.
- [ ] T035 [US1] Create `main/controllers/ReferenceDataController.java` (`@RestController`, `@RequestMapping("/api/v1/reference_data")`, `@Tag(name = "Reference Data")`, `@SecurityRequirement(name = "bearerAuth")`):
  - `@GetMapping(value = "/{attribute_name}", produces = APPLICATION_JSON_VALUE)` `getReferenceData(@PathVariable("attribute_name") String attributeName)`, which returns `ResponseEntity.ok(service.getAll(attributeName))`.
  - `@Operation(operationId = "getReferenceData", summary = "Get reference data")`.
  - `@ApiResponse` entries for `200` (`ReferenceDataApiResponse`), and `400`, `401`, `406` and `500` (`ErrorResponse`, each with the contract example message; Principle XVIII).
  - No type-specific code and no try/catch.
  - No literal type name anywhere, including annotation attributes: no `@Parameter(example = "appointment_titles")` and no `@Schema(allowableValues = …)` naming a type. The `attribute_name` enum, description and any example come only from T034's registry-driven customizer, so T061's architecture test stays green (finding U3).
- [ ] T036 [US1] Add the type declaration to `src/main/resources/application.yml`, without an alias yet (the alias arrives in T053):
  ```yaml
  jo:
    reference-data:
      types:
        - name: appointment_titles
          fixture: classpath:reference-data/appointment_titles.json
  ```
  Add `jo.security.bearer-tokens: [test-token]` to each new suite's `src/<suite>/resources/application-test.yml`, activated with `@ActiveProfiles("test")`.
- [ ] T037 [US1] Add an integration test `it/InternalFailureIntegrationTest.java` (spec EC-010, FR-026): `@MockitoBean ReferenceDataService` whose `getAll` throws `new IllegalStateException("boom /secret/path")`. `GET /api/v1/reference_data/appointment_titles` with a token must return `500` with body `error` "Internal server error.", and the body must not contain `boom`, `IllegalStateException`, `/secret/path` or `at uk.gov`. Also write a second test class, `it/NotAcceptableIntegrationTest.java`, that uses the real `ReferenceDataService` (no mock, because the `@MockitoBean` above would replace it). With a token, `GET /api/v1/reference_data/appointment_titles` with `Accept: text/plain` must return `406` with `Content-Type: application/json` and body `error` "Not acceptable.". The body must have exactly the `ErrorResponse` keys, and `traceId` must equal the `X-Correlation-Id` response header (spec EC-008, FR-025; finding F6).
- [ ] T038 [US1] Write `smoke/ReferenceDataSmokeTest.java`: one authenticated `GET /api/v1/reference_data/appointment_titles` returning `200` with 194 results, plus a p95 check: after 5 warm-up calls, 20 measured calls with p95 under **1 s** by default, a regression guard that is reliable on shared CI runners. The strict 100 ms developer-machine target applies only when the system property `-Dperf.strict=true` is set (forward it in `build.gradle`'s `smokeTest` task with `systemProperty 'perf.strict', providers.systemProperty('perf.strict').getOrElse('false')`). T067 runs the strict check once locally (research R13; finding B1).
- [ ] T039 [US1] Add a "Reference Data" folder to `postman/ctam-jomockapi.postman_collection.json` (Principle XVII):
  - Request `GET {{baseUrl}}/api/v1/reference_data/appointment_titles - success` with header `Authorization: Bearer {{bearerToken}}`.
  - Tests: status `200`, JSON content type, `results` is an array of length 194, the first record's keys equal the six contract fields, and ids are ascending.

  Add `bearerToken` = `local-dev-token` to `postman/ctam-jomockapi.postman_environment.json`.

**Checkpoint**: US1 is fully working. T028, T029 (Collection), T030, T037 and T038 pass, and the newman run passes for the collection request.

---

## Phase 4: User Story 2 — Integrator Retrieves a Single Appointment Title by ID (Priority: P1)

**Goal**: `GET /api/v1/reference_data/appointment_titles/{reference_id}` returns one record, with `400` for malformed ids, `404` for unknown ids, and `400` for any query parameter on either route.

**Independent Test**: With a token, id 70 returns the Area Coroner record, `abc` returns `400`, `15` returns `404`, and `?page=2` returns `400` (AC-004, AC-008, AC-009, AC-020).

### Tests for User Story 2 ⚠️ (write first; confirm they fail)

- [ ] T040 [P] [US2] Write `unit/services/ReferenceIdParserTest.java` (research R4):
  - Accepted: `0`, `7`, `007` (→ 7), `9223372036854775807`.
  - Rejected with `InvalidReferenceIdException`: `abc`, `12x`, `1.5`, `-1`, `+1`, ` 1`, `1 `, `1e3`, `""`, `０` (full-width digit).
  - `9223372036854775808` and a 30-digit value give "no such id" (`OptionalLong.empty()`, see T045; the service turns it into `404` in T047), not an exception.
- [ ] T041 [P] [US2] Write `unit/filters/NoQueryParametersInterceptorTest.java`: with a `HandlerMethod` handler, `preHandle` throws `UnsupportedQueryParameterException` when the query string is non-empty (`?page=2`, `?name=`, `?a`), and returns true when there is none. With a non-`HandlerMethod` handler (for example a `ResourceHttpRequestHandler`), it returns true even when there is a query string (finding C2).
- [ ] T042 [P] [US2] Add to `ft/ReferenceDataFunctionalTest.java` an `@Nested SingleRecordTests` class (US2):
  - **AC-004**: `GET …/appointment_titles/70` → `200` with a single object (no `results` key) equal to the element with id 70 from the collection response.
  - **AC-008**: `abc`, `12x`, `1.5`, `-1` → `400` "reference_id must be a non-negative whole number."
  - **AC-009**: `15` and `999999` → `404` "Reference data record not found."
  - **EC-003**: `007` resolves to 7, which is unknown, so `404`; `010` → id 10 (`200`); `99999999999999999999` → `404`.
  - **AC-020**: `?page=2` on the collection route and `?name=Judge` on `/70` → `400` "Query parameters are not supported on this endpoint."
  - **EC-002 / EC-004 / EC-005**: `…/appointment_titles/` → `404` "Resource not found."; `…/appointment_titles/70/` → `404`; `…/appointment_titles/1/extra` → `404`.
  - **Route before query (contract order of checks; finding C2)**: `…/appointment_titles/?page=2` and `…/appointment_titles/1/extra?x=1` → `404` "Resource not found.", not `400`.
  - **EC-007 (finding E1)**: with a valid token, `POST`, `PUT`, `PATCH` and `DELETE` on `…/appointment_titles` and on `…/appointment_titles/70` → `405` "Method not allowed.", in the shared `ErrorResponse` shape, with `traceId` equal to the `X-Correlation-Id` response header, and with no `results` or `id` in the body. `POST …/appointment_title` (the alias) gives the same result.
- [ ] T043 [P] [US2] Extend `unit/controllers/ReferenceDataControllerTest.java` with single-record cases: `200` for a single object, and the service called with the raw path value.
- [ ] T044 [P] [US2] Extend `it/ReferenceDataOpenApiContractTest.java` for `/api/v1/reference_data/{attribute_name}/{reference_id}`: `operationId` `getReferenceDataById`; `reference_id` is a string with `pattern` `^[0-9]+$`; responses `200` (`ReferenceDataResponse`), and `400`, `401`, `404`, `406` and `500` (`ErrorResponse`), with no other codes; `bearerAuth` required. As in T030, it asserts the optional `X-Correlation-Id` request header parameter with its `pattern`, an `X-Correlation-Id` response header on every response, and `WWW-Authenticate` on `401` (finding D2).

### Implementation for User Story 2

- [ ] T045 [US2] Create `main/services/ReferenceIdParser.java` (`@Component`) with `OptionalLong parse(String raw)`:
  - If `raw` doesn't match `^[0-9]+$` (ASCII only), throw `InvalidReferenceIdException` carrying the sanitised raw value as the diagnostic.
  - Otherwise return `OptionalLong.of(Long.parseLong(raw))`, or `OptionalLong.empty()` when the value overflows `long`. The caller treats empty as not found.
- [ ] T046 [US2] Create `main/filters/NoQueryParametersInterceptor.java` (a `HandlerInterceptor` `@Component`): `preHandle` returns true straight away unless `handler instanceof HandlerMethod`, and otherwise throws `UnsupportedQueryParameterException` if `request.getQueryString()` is non-null and non-empty. The `HandlerMethod` check is needed because `WebConfig`'s interceptor registration also applies to Spring's static-resource handler mapping. Without the check, an unmatched path such as `…/appointment_titles/?page=2` would get `400` instead of the contract's `404`, because the contract checks route before query parameters (finding C2). Register it in `main/config/WebConfig.java` `addInterceptors` for the path pattern `/api/v1/reference_data/**` (research R3). The handler's `WARN` log line includes the sanitised query string.
- [ ] T047 [US2] Add `ReferenceDataResponse getById(String attributeName, String rawReferenceId)` to `main/services/ReferenceDataService.java`: resolve the type first, then parse the id, then look it up. Throw `ReferenceDataNotFoundException` for an empty `OptionalLong` or a missing record, and otherwise map the record. Extend `unit/services/ReferenceDataServiceTest.java` for: found, not found, overflow → not found, malformed → exception, and an unsupported type checked before the id.
- [ ] T048 [US2] Add to `main/controllers/ReferenceDataController.java` `@GetMapping(value = "/{attribute_name}/{reference_id}", produces = APPLICATION_JSON_VALUE)` `getReferenceDataById(@PathVariable("attribute_name") String attributeName, @Parameter(schema = @Schema(type = "string", pattern = "^[0-9]+$")) @PathVariable("reference_id") String referenceId)`, with `@Operation(operationId = "getReferenceDataById", summary = "Get reference data by id")` and `@ApiResponse` entries for `200`, `400`, `401`, `404`, `406` and `500` (Principle XVIII). As in T035, no literal type name appears in any annotation. A `reference_id` example of `"70"` is fine (finding U3).
- [ ] T049 [US2] Add these Postman requests to the "Reference Data" folder, each with status, content-type and body assertions using the contract's messages:
  - `GET …/appointment_titles/70 - success` (asserts `name` "Area Coroner" and `end_date` "2025-03-31")
  - `GET …/appointment_titles/abc - malformed id` (`400`)
  - `GET …/appointment_titles/15 - unknown id` (`404`)
  - `GET …/appointment_titles?page=2 - query parameter rejected` (`400`)
  - `GET …/appointment_titles/ - trailing slash` (`404`)

**Checkpoint**: US1 and US2 both work. The single-record, validation and query-parameter scenarios pass.

---

## Phase 5: User Story 5 — Unauthenticated or Invalidly Authenticated Requests Are Refused (Priority: P1)

**Goal**: Every reference-data request without a valid Bearer token gets `401`. Tokens are configurable but authentication can't be turned off, and the healthcheck stays public. (The filter was built in T020; this phase proves the story end to end.)

**Independent Test**: With no token, a wrong token or an empty token, every route returns `401` with `WWW-Authenticate: Bearer`. A changed configured token is honoured, an empty token set stops startup, and the healthcheck returns `200` without a token (AC-010, AC-011, AC-017, AC-018).

### Tests for User Story 5 ⚠️

- [ ] T050 [P] [US5] Add to `ft/ReferenceDataFunctionalTest.java` an `@Nested AuthenticationTests` class (US5):
  - **AC-010**: with no `Authorization` header, each of these returns `401` with the exact message, `WWW-Authenticate: Bearer`, and no `results` or `id` in the body: `…/appointment_titles`, `…/appointment_titles/70`, `…/genders`, `…/appointment_titles/abc`, `…/appointment_titles?page=2`, `…/appointment_titles/`, and `POST …/appointment_titles`. This checks that authentication runs first (spec EC-009, EC-013).
  - **AC-011**: repeat the collection and single-record routes with `Basic dGVzdA==`, `Bearer`, `Bearer ` (empty), `Bearer wrong`, and `Bearer test-token ` (trailing space). All give the same `401` body message.
  - **AC-018**: `GET /api/v1/healthcheck` with no header → `200` `{"status":"ok"}`.
  - **SC-005**: none of these `401` bodies contains reference data.
  - **Encoded-path bypass (finding C1, Principle XV)**: over real HTTP (Tomcat), with no token, each of these returns `401` and never `200`:
    - `/%61pi/v1/reference_data/appointment_titles`
    - `/api/v1/%72eference_data/appointment_titles`
    - `/./api/v1/reference_data/appointment_titles`
    - `/api;x=1/v1/reference_data/appointment_titles`

    Also, `/api/v1/%68ealthcheck` without a token still returns `200` (it is the healthcheck). Build each request with `URI.create(...)` so `TestRestTemplate` doesn't re-encode `%` as `%25`. If Tomcat rejects a form outright with `400`, that is acceptable. The only failing outcome is a `200` carrying reference data.
- [ ] T051 [P] [US5] Write `it/AuthenticationConfigurationIntegrationTest.java`:
  - **AC-017**: `@SpringBootTest` with `jo.security.bearer-tokens=rotated-token`. `rotated-token` → `200`; `test-token` and `local-dev-token` → `401`.
  - **Two tokens accepted**: `jo.security.bearer-tokens[0]=a`, `[1]=b`; both return `200`.
  - **Can't be turned off**: use `ApplicationContextRunner` (or `SpringApplicationBuilder` in a `assertThatThrownBy`) to show that an empty list (`jo.security.bearer-tokens=`), or a list containing a blank entry, makes context startup fail with the T019 message.
  - **Profile isolation (finding C1)**: the environment-override and empty-token cases below MUST NOT activate the `test` profile. `application-test.yml` (T036) sets `jo.security.bearer-tokens` directly, which overrides the `${JO_SECURITY_BEARER_TOKENS:…}` placeholder in `application.yml`, so under that profile the variable would never be read. Put these cases in their own `@Nested` class, or a separate test class, with no `@ActiveProfiles`. Each one also asserts that `test-token` → `401`, which proves the profile's list isn't in effect.
  - **Environment override (finding F1)**: with `JO_SECURITY_BEARER_TOKENS=env-a,env-b` supplied as a property source (`@TestPropertySource(properties = "JO_SECURITY_BEARER_TOKENS=env-a,env-b")` resolves the same placeholder an environment variable would), `env-a` and `env-b` → `200` and `local-dev-token` → `401`. With `JO_SECURITY_BEARER_TOKENS=` (empty), startup fails with the T019 message.
  - **Token never logged**: with `@ExtendWith(OutputCaptureExtension.class)`, send `Bearer super-secret-value` and `Bearer test-token`, then assert the captured output contains neither `super-secret-value` nor `test-token`.

### Implementation for User Story 5

- [ ] T052 [US5] Add these Postman requests to the "Reference Data" folder, each asserting `401`, the exact message and the `WWW-Authenticate` header:
  - `GET …/appointment_titles - missing token` (no auth header)
  - `GET …/appointment_titles - invalid token` (`Authorization: Bearer {{invalidBearerToken}}`)

  Add `invalidBearerToken` = `not-a-valid-token` to `postman/ctam-jomockapi.postman_environment.json`. If T050 or T051 exposed defects in T019 or T020, fix them in `main/filters/BearerTokenAuthenticationFilter.java` or `main/config/SecurityProperties.java`.

**Checkpoint**: All P1 stories (US1, US2, US5) are complete and form the shippable MVP.

---

## Phase 6: User Story 3 — Legacy Consumer Uses the Deprecated Alias (Priority: P2)

**Goal**: `appointment_title` behaves exactly like `appointment_titles` on both routes, for success, not-found and validation failures, with no sign that an alias was used.

**Independent Test**: For each pair of requests that differs only by attribute name, status, content type and body are identical (AC-002, AC-003, AC-005, AC-006, AC-016; SC-002).

### Tests for User Story 3 ⚠️

- [ ] T053 [P] [US3] Add to `ft/ReferenceDataFunctionalTest.java` an `@Nested AliasTests` class (US3), using a JUnit `@ParameterizedTest` over path suffixes `""`, `/10`, `/70`, `/15`, `/abc`, `?page=2`:
  - **AC-002 / AC-003 / AC-005 / AC-006 / SC-002**: `/appointment_titles<suffix>` and `/appointment_title<suffix>` give equal status and `Content-Type`. Bodies are equal once `timestamp` and `traceId` are stripped from error bodies, and success bodies are equal byte for byte.
  - **AC-016**: the response header names are the same set for both, apart from values of `X-Correlation-Id` and `Date`. No success body contains the string `appointment_title`.

### Implementation for User Story 3

- [ ] T054 [US3] Add `aliases: [appointment_title]` to the `appointment_titles` entry in `src/main/resources/application.yml` (spec FR-003). Update the expected `enum` in `it/ReferenceDataOpenApiContractTest.java` (T030) to `["appointment_titles", "appointment_title"]`, and assert that the parameter description mentions `appointment_title` as deprecated (research R10).
- [ ] T055 [US3] Add these Postman requests to the "Reference Data" folder, each with a test asserting `200` and the same `results` length or `id` as the canonical request:
  - `GET …/appointment_title - deprecated alias`
  - `GET …/appointment_title/70 - deprecated alias`

**Checkpoint**: The alias works on both routes and matches the canonical name in every tested case.

---

## Phase 7: User Story 4 — Unsupported Reference-Data Types Are Rejected (Priority: P2)

**Goal**: Every attribute name other than the two supported values returns `400` and never returns data. That includes the 20 names the contract allows but this feature doesn't serve, unknown names and case variants.

**Independent Test**: Every one of the 20 other E-Links names, plus `foo` and case variants, returns `400` on both routes (AC-007, AC-014; SC-004; EC-001).

### Tests for User Story 4 ⚠️

- [ ] T056 [P] [US4] Add to `ft/ReferenceDataFunctionalTest.java` an `@Nested UnsupportedTypeTests` class (US4), using `@ParameterizedTest` `@ValueSource`/`@MethodSource` over:
  - the 20 E-Links names: `base_locations`, `contract_types`, `genders`, `judiciary_roles`, `jurisdictions`, `location_types`, `locations`, `ticket_categories`, `ticket_category_types`, `tickets`, `base_location`, `contract_type`, `gender`, `judiciary_role`, `jurisdiction`, `location_type`, `location`, `ticket_category`, `ticket_category_type`, `ticket`
  - `foo`, `Appointment_Titles`, `APPOINTMENT_TITLES`, `appointment_titles_`

  For each name, `GET /{name}` and `GET /{name}/10` give `400` with "Unsupported reference data attribute_name.", and the body has no `results` or `id` (AC-007, AC-014, SC-004, EC-001). Also, `GET /foo/abc` gives the unsupported-type message, because the type is checked before the id (research R3).
- [ ] T057 [P] [US4] Extend `unit/controllers/ReferenceDataControllerTest.java`: when the mocked service throws `UnsupportedReferenceDataTypeException`, the response is `400` with the shared shape.

### Implementation for User Story 4

- [ ] T058 [US4] Add these Postman requests to the "Reference Data" folder, each asserting `400` and the exact message:
  - `GET …/genders - unsupported (contract-valid, out of scope)`
  - `GET …/foo/1 - unsupported (unknown)`

  If T056 fails, fix it in `main/services/ReferenceDataTypeRegistry.java` or `main/exceptions/GlobalExceptionHandler.java`. Don't add special cases to the controller.

**Checkpoint**: None of the other types is served, and the rejection is the same on both routes.

---

## Phase 8: User Story 6 — Maintainer Can Add Further Reference-Data Types Without Duplicating Endpoint Logic (Priority: P3)

**Goal**: Show that a new type can be added through configuration plus a fixture alone, and that nothing in the main code is specific to AppointmentTitle.

**Independent Test**: A `test_widgets` type declared only in test properties is served through the same routes with the same validation, authentication and errors, and no main source file mentions appointment titles (AC-013, AC-019; SC-007; EC-011).

### Tests for User Story 6 ⚠️

- [ ] T059 [P] [US6] Create the test fixtures:
  - `src/integrationTest/resources/reference-data/test_widgets.json`: three records with ids 5, 9 and 3 (unsorted), one with a non-null `end_date`, all following data-model.md §2
  - `src/integrationTest/resources/reference-data/empty_things.json`: `[]`
- [ ] T060 [US6] Write `it/ReferenceDataExtensionIntegrationTest.java`: `@SpringBootTest` + MockMvc, with `@TestPropertySource` declaring the full `jo.reference-data.types` list (`appointment_titles` with its alias, `test_widgets` with alias `test_widget` and fixture `classpath:reference-data/test_widgets.json`, and `empty_things` with fixture `classpath:reference-data/empty_things.json`). Assert:
  - **AC-019**: `/test_widgets` → `200` with ids `[3, 5, 9]`; `/test_widget/9` → `200`; `/test_widgets/4` → `404`; `/test_widgets/x` → `400`; no token → `401`; `?a=1` → `400`.
  - **EC-011**: `/empty_things` → `200` `{"results":[]}`; `/empty_things/1` → `404`.
  - `/v3/api-docs` lists all five names in the `attribute_name` `enum`.
  - An invalid declaration (the name `appointment_titles` reused as an alias of `test_widgets`) makes startup fail (`ApplicationContextRunner`).
- [ ] T061 [P] [US6] Write `unit/architecture/NoTypeSpecificCodeTest.java` (AC-013, FR-005): walk `src/main/java` and assert that no `.java` file contains `appointment` (case-insensitive) anywhere, **including comments and Javadoc**. Write type-neutral examples in comments instead (e.g. "e.g. `<attribute_name>`"; finding U4). Also assert that only `ReferenceDataTypeRegistry.java` looks up an attribute name or alias. For the second check, no other file under `main/` may reference `getAliases()` or `.aliases()`.

### Implementation for User Story 6

- [ ] T062 [US6] Add a "Reference data" section to `README.md` covering: the endpoints, the Bearer token (`local-dev-token` is non-secret; override with `JO_SECURITY_BEARER_TOKENS`), and **"Adding a reference-data type"**, which is one `jo.reference-data.types` entry plus one fixture following data-model.md §2, with no Java changes (FR-008). Link to `specs/002-reference-data-api/`.

**Checkpoint**: All six user stories are complete.

---

## Phase 9: Polish & Cross-Cutting Concerns

- [ ] T063 [P] Write `it/LoggingIntegrationTest.java` using `OutputCaptureExtension` (Principles IX and XIV):
  - A `400` (`/foo` with `X-Correlation-Id: log-123`) logs at `WARN` and the JSON line contains `"correlationId":"log-123"`.
  - A `500` (as in T037) logs at `ERROR` with the correlation ID.
  - A request with `X-Correlation-Id: a%0D%0Ainjected` gets a generated ID, and the captured output has no line starting `injected`.
  - A path segment containing encoded CRLF (`/foo%0D%0AFAKE`) is logged sanitised. Tomcat may reject this request itself with `400` before it reaches the application. That outcome is acceptable, and so is the application's `400` with a sanitised log line. The test fails only if the captured output contains a line starting `FAKE` (finding B1).
- [ ] T064 [P] Update `CLAUDE.md` § "Current repo state":
  - Replace the greenfield text with the current state: the healthcheck and reference-data features, the four test suites, and `./gradlew check [-PskipOwasp]`.
  - Correct § "Reference data": the extract lives in the sibling checkout `../ctam-jomockapi/joh-elinks-api/`, not in this repo.
  - Add the suite commands to § Commands.
- [ ] T065 [P] Update the "Status" section of `README.md` to list the Reference Data API (appointment titles) alongside the healthcheck, and document the local quality-gate commands from quickstart.md §1.
- [ ] T066 Run `./gradlew check` with `NVD_API_KEY` set (or `-PskipOwasp` if no key is available, and record that) and confirm: zero compiler warnings, Checkstyle clean, all four suites green and the JaCoCo report produced. Record the line coverage for the new packages. Fix any OWASP finding of CVSS ≥ 7, or suppress it in `config/owasp/suppressions.xml` with a documented reason. Re-run `./gradlew dependencies --write-locks` if dependencies changed.
- [ ] T067 Run `./gradlew smokeTest -Dperf.strict=true` once locally and record the result (research R13). Start `./gradlew bootRun` and run every step of `specs/002-reference-data-api/quickstart.md` §§2–6, including the empty-token startup refusal. Then run `npx newman run postman/ctam-jomockapi.postman_collection.json -e postman/ctam-jomockapi.postman_environment.json` and confirm every Healthcheck and Reference Data request passes. Check Swagger UI by hand: authorise with `local-dev-token` and try both operations (SC-006).
- [ ] T068 Review the finished code against the plan's Constitution Check table. In particular, confirm: no `Map<String, Object>` in `main/`; the controller does nothing but delegate; there is no try/catch in controllers; `/v3/api-docs` documents no endpoint that isn't implemented (XVIII); and every Postman request matches the contract (XVII). Confirm that the Sonar deferral issue from T006 exists and is linked from the PR description, or that the guard was left out because `SONAR_TOKEN` was already configured (XIII). Record the result in a short "Implementation notes" section at the end of `specs/002-reference-data-api/plan.md`.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: no dependencies. T001 → T002 → T003 → T004 → T005 → T006 → T007 run in order, because they all edit `build.gradle`. T008 and T009 can run in parallel at any point.
- **Foundational (Phase 2)**: depends on T007. **Blocks all stories.**
- **US1 (Phase 3)**: depends on Phase 2. **The MVP core.**
- **US2 (Phase 4)**: depends on Phase 2. Also depends on US1's controller, service and DTO files (T031, T033, T035), because it extends them. Its tests can be written in parallel with US1.
- **US5 (Phase 5)**: depends on Phase 2 (T019, T020), plus at least T035 so there is a route to authenticate against, and T048 for the single-record cases.
- **US3 (Phase 6)**: depends on US1 and US2, because it compares both routes.
- **US4 (Phase 7)**: depends on US1 and US2, because it tests both routes.
- **US6 (Phase 8)**: depends on US1 and US2. It is independent of US3, US4 and US5 apart from sharing `application.yml`.
- **Polish (Phase 9)**: depends on all stories.

### Within Each Story

Write the tests first and confirm they fail. Then build DTOs/entities, then services, then the controller and configuration, then Postman. Each story ends at its checkpoint.

### Shared-File Hot Spots (never [P] against each other)

These files are edited by several stories, so tasks that touch the same one must run in order:

| File | Edited by |
|------|-----------|
| `build.gradle` | T001–T007, T038 (adds the `perf.strict` system property to the `smokeTest` task) |
| `application.yml` | T016, T018, T019, T036, T054 |
| `ReferenceDataController.java` | T035, T048 |
| `ReferenceDataService.java` | T033, T047 |
| `ReferenceDataFunctionalTest.java` | T029, T042, T050, T053, T056 (one `@Nested` class per story, so conflicts are small, but add them in order) |
| Postman collection | T039, T049, T052, T055, T058 |

---

## Parallel Examples

### Phase 2 (after T007)

```text
T010 ErrorResponse record          T011 exception classes
T012 LogSanitiser + test           T021 entities
T022 ReferenceDataProperties       T025 DTOs + mapper
```

Then T013 → T014 → T015/T016/T017 (T017 is parallel with T015 and T016), then T019 → T020, then T023 → T024, then T026 and T027.

### User Story 1

```text
T028 controller slice test      T029 functional collection tests      T030 OpenAPI contract test
T031 DTO @Schema docs
```

Then T032 (fixture), then T033 → T034 → T035 → T036, then T037, T038 and T039.

### User Story 2

```text
T040 parser test   T041 interceptor test   T042 functional tests   T043 controller test   T044 contract test
```

Then T045 and T046 in parallel, then T047 → T048 → T049.

### User Story 5

```text
T050 functional auth tests      T051 auth configuration integration tests
```

---

## Implementation Strategy

### MVP (P1 stories: US1 + US2 + US5)

1. Phase 1 (Setup): quality gates and suites green on the existing code.
2. Phase 2 (Foundational): error shape, correlation IDs, logging, authentication, registry, repository and mapper.
3. Phase 3 (US1). **Stop and check**: 194 titles come back with a token.
4. Phase 4 (US2): single record, validation and query-parameter rejection.
5. Phase 5 (US5): authentication proven end to end. **This is the MVP**: a secured, contract-shaped appointment-titles API.

### Incremental delivery after the MVP

6. US3 (alias), then US4 (unsupported types), then US6 (extensibility proof and README).
7. Phase 9: logging checks, docs, the full `check` with OWASP, quickstart, newman, and the constitution review.

### Commits

Commit at the end of each phase checkpoint, at least. Phase 1 on its own should be one reviewable commit ("build: quality gates and test suites"), because it touches no feature code.

---

## Notes

- **68 tasks.** Every acceptance scenario is covered by at least one automated test:

  | Scenarios | Covered by |
  |-----------|------------|
  | AC-001, AC-012, AC-015 | T029 |
  | AC-002, AC-003, AC-005, AC-006, AC-016 | T053 |
  | AC-004, AC-008, AC-009, AC-020 | T042 |
  | AC-007, AC-014 | T056 |
  | AC-010, AC-011, AC-018 | T050 |
  | AC-017 | T051 |
  | AC-013 | T061 |
  | AC-019 | T060 |

- **Edge cases**:

  | Edge case | Covered by |
  |-----------|------------|
  | EC-001 | T056 |
  | EC-002, EC-004, EC-005 | T042 |
  | EC-003 | T040, T042 |
  | EC-006 | T041, T042 |
  | EC-007 | T042 (authenticated `405`), T027, T050 (unauthenticated → `401`) |
  | EC-008 (406) | T015 (handler unit test), T037 (end to end) |
  | EC-009, EC-013 | T027, T050 |
  | EC-010 | T037 |
  | EC-011 | T060 |
  | EC-012 | T028, T029 |

- The fixture-generation script is deliberately not committed. data-model.md holds the rules, and the checked-in JSON is the source of truth (research R6).
