# Implementation Plan: Reference Data API — Appointment Titles

**Branch**: `002-reference-data-api` | **Date**: 2026-09-23 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/002-reference-data-api/spec.md`

**Note**: This template is filled in by the `/speckit-plan` command; its definition describes the execution workflow.

## Summary

This feature adds `GET /api/v1/reference_data/{attribute_name}` and `GET /api/v1/reference_data/{attribute_name}/{reference_id}`. Both are served by one generic reference-data mechanism that is driven by configuration.

**What is served**:
- Only `appointment_titles` is configured, with its deprecated alias `appointment_title`.
- Attribute names and aliases are resolved centrally by `ReferenceDataTypeRegistry`.
- Records come from a checked-in, synthetic, deterministic fixture of 194 titles, loaded and validated at startup.
- Records are mapped to typed DTOs with MapStruct. The response shape follows the E-Links Swagger, plus `name` (deviation D-1).

**Shared infrastructure built here**: because this is the first business endpoint, it also builds what `specs/001-healthcheck-api` deferred (research R1):
- centralised, configurable Bearer-token authentication, which can't be turned off
- correlation-ID handling
- structured logging with sanitised caller input
- the shared `ErrorResponse` / `GlobalExceptionHandler`, which also covers framework 404/405/406/500
- the Principle XIII quality gates and the four test suites

## Technical Context

**Language/Version**: Java 25 (Gradle toolchain), per Principle II.

**Build Tool**: Gradle 9.7.1 via the wrapper. Dependency locking stays on: `gradle.lockfile` is regenerated with `--write-locks`.

**Primary Dependencies**:
- Already present: Spring Boot 4.1.1 (`spring-boot-starter-web`, which uses Jackson 3 for MVC) and springdoc-openapi 3.1.1.
- **New**:
  - Lombok 1.18.48
  - MapStruct 1.6.3, with `lombok-mapstruct-binding` 0.2.0
  - OWASP Java Encoder 1.4.0
- **New build plugins**:
  - `uk.gov.hmcts.java` 0.12.70 (Checkstyle and OWASP dependency-check)
  - JaCoCo 0.8.15
  - `org.sonarqube` 7.5.0.8588
  - `com.github.ben-manes.versions` 0.64.0

  Fallbacks are in research R11.

**Storage**: No database. One JSON fixture per reference-data type, on the classpath, loaded into immutable in-memory maps at startup (research R5, R6).

**Testing**: JUnit 5, AssertJ and MockMvc, split into four Gradle JVM test suites: `test` (unit and `@WebMvcTest`), `integrationTest`, `functionalTest` and `smokeTest` (research R11). OpenAPI contract tests assert `/v3/api-docs` against [contracts/reference-data-api.md](./contracts/reference-data-api.md). The Postman collection is run with newman.

**Target Platform**: JVM 25 as an executable Spring Boot JAR. CI on GitHub Actions (`ubuntu-latest`, Temurin 25).

**Project Type**: A single web service.

**Performance Goals**: p95 under 100 ms for both routes on a developer machine. Each call is an in-memory lookup over 194 records (research R13).

**Constraints**:
- Authentication can't be turned off.
- No query parameters are accepted.
- Routes match exactly.
- Error bodies never echo caller input or tokens.
- Responses are deterministic for a given fixture.
- Builds fail on any compiler warning.

**Scale/Scope**: One reference-data type (194 records) and two routes. The design must let 10 more types be added through configuration only (spec FR-008, SC-007).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-checked after Phase 1 design below.*

| Principle | Assessment |
|---|---|
| I. Contract-First Development | **Pass, with one agreed deviation.** Routes (apart from the project-wide `/api/v1` base), path parameters, the `results` wrapper, record fields, media type and `401` all follow the E-Links Swagger. D-1 (the added `name` field) was agreed in spec Clarifications and recorded in spec § Contract Deviations; see Complexity Tracking. The status codes the contract leaves out (`400`/`404`/`405`/`406`/`500`) fill gaps and contradict nothing (spec Assumptions). |
| II. Java Spring Boot Architecture | **Pass.** Java 25, Spring Boot 4.1.1, Gradle. Code is organised by layer under `uk.gov.hmcts.ctam.jo`. `ReferenceDataController` only binds the request and shapes the response; resolution, parsing and lookup are in `services/`. `GlobalExceptionHandler` is the single `@RestControllerAdvice` in `exceptions/`. One small `util/` package holds `LogSanitiser`; the principle's package list is illustrative ("such as"). `JsonErrorController` (the `/error` fallback) sits in `exceptions/` rather than `controllers/` on purpose. It serves no business route and exists only so that errors raised outside Spring MVC get the same `ErrorResponse` body as `GlobalExceptionHandler` produces. Keeping it next to the handler and `ErrorResponseFactory` keeps all error rendering in the one layer package Principle II assigns to it. |
| III. Reuse Before Duplication | **Pass.** One controller, service, registry, repository and mapper serve every reference-data type. The shared error writer is used by the handler, the filter and the `/error` controller. The query-parameter rule is one interceptor, and `reference_id` parsing is one parser. |
| IV. Synthetic Data Only | **Pass.** Only the public title *names* come from the extract (Clarification 2026-09-23). Ids, timestamps and dates are generated by the rules in data-model.md, and no other extract data is used. |
| V. Deterministic Behaviour | **Pass.** A static fixture, output sorted by `id`, a fixed JSON property order, fixed error messages, and an injected `Clock`. Nothing random appears on any response path; UUID correlation IDs are generated only when the caller didn't supply one, and only affect `traceId`. |
| VI. Behavioural Fidelity | **Pass.** No parameter is accepted and then ignored: undefined query parameters are rejected with `400` (FR-027). The contract defines no filtering or pagination for reference data, so there is none. |
| VII. Generic Reference-Data Handling | **Pass.** Types come from configuration. `ReferenceDataTypeRegistry` is the only place aliases are resolved. Adding a type means adding configuration and a fixture, with no endpoint code (AC-019 test). |
| VIII. Typed API Models | **Pass.** Record DTOs in `domain/`, a separate `entity/` model, a MapStruct mapper, Lombok for the entity and components, and no `Map<String, Object>` anywhere. |
| IX. Centralised Validation and Error Handling | **Pass (closes 001's deferral).** One `ErrorResponse(error, timestamp, traceId)` is produced by one factory for every error the application returns: application exceptions, framework 404/405/406, errors raised in filters (401) and errors outside MVC (the `/error` controller). Authentication is simulated, centralised and configurable. 4xx are logged at `WARN`, 5xx at `ERROR`. |
| X. Configuration Over Hard-Coding | **Pass.** Tokens, type declarations and fixture locations are externalised (`jo.security.*`, `jo.reference-data.*`). The healthcheck authentication exemption is deliberately *not* configurable, so authentication can't be turned off (Clarification). That is a security constraint, not environment-specific behaviour. |
| XI. Testability | **Pass.** Unit tests cover the registry, parser, service, mapper, repository validation, filters, interceptor, handler and sanitiser. Controller tests use `@WebMvcTest`. Integration tests cover the full context, including the OpenAPI contract assertions. The functional suite covers AC-001 to AC-020. A smoke suite is included. |
| XII. Simplicity and Maintainability | **Pass.** A servlet filter rather than Spring Security, configuration-declared types rather than per-type provider beans, and a static fixture rather than a generator (research R2, R5, R6). No abstraction is added without at least two call sites, apart from the `ReferenceDataRepository` interface, which lets unit tests replace the fixture loader. |
| XIII. Automated Quality Gates | **Pass (closes 001's deferral), with one documented deferral and one conditional risk.** **Deferral**: the Sonar step in CI stays inactive until a SonarQube Cloud admin switches off Automatic Analysis and adds `SONAR_TOKEN`; see Complexity Tracking for the way back to compliance. `-Werror`, Checkstyle, Sonar, OWASP dependency-check (suppressions in `config/owasp/`), `dependencyUpdates`, JaCoCo, the four suites, `ci.yml` running `./gradlew check`, and a separate `codeql.yml`. **Conditional**: if CodeQL can't analyse Java 25 yet, record it in Complexity Tracking with the way back to compliance (research R11, risk 2). |
| XIV. Observability & Traceability | **Pass.** `X-Correlation-Id` is read (format checked) or generated, stored in MDC under `correlationId`, echoed in the response header and `traceId`, and included in every structured log line. Caller values are sanitised with OWASP Encoder before logging. Only the healthcheck is exempt, as before. |
| XV. Security by Design | **Pass. Decisions recorded here:** (1) Bearer tokens are compared in constant time and never logged. (2) Startup fails if the token set is empty. (3) Authentication runs before routing, so unauthenticated callers can't probe routes. (4) The inbound correlation header is format-checked before it is echoed or logged, which blocks injection. (5) Error bodies are generic and contain no internals. (6) **Relaxed on purpose**: `/v3/api-docs` and `/swagger-ui/**` are unauthenticated. They document a public mock contract, hold no data, and sit outside `/api/`. The healthcheck exemption is unchanged. (7) The default token `local-dev-token` is clearly marked as non-secret, and deployments override it. |
| XVI. Object-Oriented Design Discipline | **Pass.** Each class has one job (resolve, parse, load, map, authenticate, correlate, render errors). New types are added without modifying code. Collaborators are injected by constructor. The controller calls only `ReferenceDataService`, which returns DTOs, so there is no call chaining through other objects. |
| XVII. API Testability and Postman Artifacts | **Pass (planned).** A "Reference Data" folder is added to `postman/ctam-jomockapi.postman_collection.json`. It holds requests for the success and alias cases on both routes, `400` (type, id, query), `401` (missing and invalid), `404` and trailing slash. Each has tests. The environment gains `bearerToken` and `invalidBearerToken`. |
| XVIII. API Contract and Swagger/OpenAPI | **Pass (planned).** springdoc annotations, the `bearerAuth` scheme, the `attribute_name` enum filled from the registry, the request and response headers (the optional `X-Correlation-Id` request header with its pattern, `X-Correlation-Id` on every response, and `WWW-Authenticate` on `401`) added once by the registry-driven customizer, `@Schema` descriptions and examples on every DTO field, and every status code documented. Contract tests assert the generated document against contracts/reference-data-api.md. |

**Non-principle checks**:
- `.editorconfig` already exists.
- The work stays transparent because every design decision is recorded in this `specs/` directory.
- **Found while planning**: CLAUDE.md § "Current repo state" still says the repo is greenfield with no build file, and says `joh-elinks-api/` is in the repo, which it isn't. A documentation task should correct both.

**Result**: No blocking violations. The one deviation (D-1) is agreed and recorded.

## Project Structure

### Documentation (this feature)

```text
specs/002-reference-data-api/
├── plan.md                       # This file
├── research.md                   # Phase 0: decisions R1–R13
├── data-model.md                 # Phase 1: types, records, DTOs, fixture rules
├── quickstart.md                 # Phase 1: run and check guide
├── contracts/
│   └── reference-data-api.md     # Phase 1: HTTP contract and error messages
├── checklists/
│   └── requirements.md           # From /speckit-specify
└── tasks.md                      # Phase 2 (/speckit-tasks; not created here)
```

### Source Code (repository root)

```text
build.gradle                                  # + plugins, deps, -Werror, test suites, JaCoCo, Sonar
gradle.lockfile                               # regenerated
config/
├── owasp/suppressions.xml                    # NEW (initially no suppressions)
└── checkstyle/checkstyle.xml                 # NEW only if the fallback route in research R11 is used
.github/workflows/
├── ci.yml                                    # NEW: ./gradlew check (+ sonar, dependencyUpdates)
└── codeql.yml                                # NEW: scheduled + PR/push SAST
postman/
├── ctam-jomockapi.postman_collection.json    # + "Reference Data" folder
└── ctam-jomockapi.postman_environment.json   # + bearerToken, invalidBearerToken
CLAUDE.md                                     # correct the stale "Current repo state" section

src/main/java/uk/gov/hmcts/ctam/jo/
├── Application.java
├── config/
│   ├── ReferenceDataProperties.java          # jo.reference-data.types[*]
│   ├── SecurityProperties.java               # jo.security.bearer-tokens (non-empty, checked at startup)
│   ├── WebConfig.java                        # registers the interceptor; Clock bean
│   └── OpenApiConfig.java                    # bearerAuth scheme; attribute_name enum customizer
├── controllers/
│   ├── HealthcheckController.java            # unchanged
│   └── ReferenceDataController.java          # the two GET routes, no type-specific code
├── domain/
│   ├── HealthResponse.java                   # unchanged
│   ├── ReferenceDataApiResponse.java         # record(results)
│   └── ReferenceDataResponse.java            # record(id, name, created_at, …)
├── entity/
│   ├── ReferenceDataRecord.java              # Lombok @Value @Builder @Jacksonized
│   └── ReferenceDataType.java                # Lombok @Value (name, aliases, fixture)
├── exceptions/
│   ├── ErrorResponse.java                    # record(error, timestamp, traceId)
│   ├── ErrorResponseFactory.java             # builds and writes the shared body (handler, filter, /error)
│   ├── GlobalExceptionHandler.java           # the single @RestControllerAdvice
│   ├── JsonErrorController.java              # /error fallback in the shared shape
│   ├── UnsupportedReferenceDataTypeException.java
│   ├── InvalidReferenceIdException.java
│   ├── ReferenceDataNotFoundException.java
│   └── UnsupportedQueryParameterException.java
├── filters/
│   ├── CorrelationIds.java                   # header name, MDC key, request-attribute name
│   ├── RequestPaths.java                     # decoded, normalised request path for filter decisions
│   ├── CorrelationIdFilter.java              # highest precedence; skips the healthcheck
│   ├── BearerTokenAuthenticationFilter.java  # /api/** except the healthcheck
│   └── NoQueryParametersInterceptor.java     # /api/v1/reference_data/**
├── mappers/
│   └── ReferenceDataMapper.java              # MapStruct
├── repository/
│   ├── ReferenceDataRepository.java          # findAll(type), findById(type, id)
│   └── FixtureReferenceDataRepository.java   # loads and validates fixtures at startup
├── services/
│   ├── ReferenceDataService.java
│   ├── ReferenceDataTypeRegistry.java        # the ONE place names and aliases are resolved
│   └── ReferenceIdParser.java
└── util/
    └── LogSanitiser.java                     # OWASP Encoder wrapper

src/main/resources/
├── application.yml                           # + jo.security, jo.reference-data, structured logging
└── reference-data/appointment_titles.json    # 194 synthetic records (data-model.md)

src/test/java/uk/gov/hmcts/ctam/jo/                # unit suite
├── controllers/HealthcheckControllerTest.java     # + @Import of the web config (research R12)
├── controllers/ReferenceDataControllerTest.java   # @WebMvcTest
├── services/…Test.java                            # registry, parser, service
├── repository/FixtureReferenceDataRepositoryTest.java
├── mappers/ReferenceDataMapperTest.java
├── filters/…Test.java                             # correlation, auth, interceptor, RequestPaths
├── exceptions/GlobalExceptionHandlerTest.java
├── exceptions/ErrorResponseFactoryTest.java
├── architecture/NoTypeSpecificCodeTest.java       # AC-013: no type names in main code
└── util/LogSanitiserTest.java
src/test/resources/reference-data/…               # small valid and invalid fixtures for the repository tests
src/integrationTest/java/uk/gov/hmcts/ctam/jo/
├── HealthcheckOpenApiContractTest.java            # moved from src/test
├── ReferenceDataOpenApiContractTest.java
├── ErrorShapeIntegrationTest.java                 # framework 404/405/406/500, check order
├── ErrorDispatchIntegrationTest.java              # /error keeps traceId and X-Correlation-Id (real HTTP)
├── InternalFailureIntegrationTest.java            # EC-010: 500 exposes no internals
├── NotAcceptableIntegrationTest.java              # EC-008: 406 in the shared shape
├── LoggingIntegrationTest.java                    # WARN/ERROR levels, correlationId, log-injection sanitising
├── AuthenticationConfigurationIntegrationTest.java  # AC-017, startup fails on an empty token set
└── ReferenceDataExtensionIntegrationTest.java     # AC-019, a test_widgets type from test properties
src/integrationTest/resources/
├── application-test.yml                           # test token
└── reference-data/{test_widgets,empty_things}.json
src/functionalTest/java/uk/gov/hmcts/ctam/jo/
└── ReferenceDataFunctionalTest.java               # AC-001 … AC-020 over real HTTP
src/functionalTest/resources/
├── application-test.yml
└── golden/appointment_titles.json                 # captured response body (AC-015)
src/smokeTest/resources/application-test.yml
src/smokeTest/java/uk/gov/hmcts/ctam/jo/
├── HealthcheckSmokeTest.java                      # moved from src/test
└── ReferenceDataSmokeTest.java
```

**Structure Decision**: A single Gradle/Spring Boot project, organised by layer under `uk.gov.hmcts.ctam.jo`, following 001's structure. This feature creates the `config/`, `entity/`, `exceptions/`, `filters/`, `mappers/`, `repository/` and `services/` packages that 001 left absent, plus `util/` for the one cross-cutting helper. Test suites get their own source sets (`src/<suite>/java`) through Gradle's JVM Test Suite plugin. The two existing `@SpringBootTest` healthcheck tests move to the suites that fit them.

## Complexity Tracking

> Deviations from constitution MUSTs, each documented per Governance.

| Item | Why Needed | Simpler Alternative Rejected Because |
|------|------------|--------------------------------------|
| **D-1**: `name` field added to `ReferenceDataResponse` (Principle I) | Agreed in spec Clarifications on 2026-09-23. Without `name`, consumers can't turn an id into a title, and the reference extract shows the real data carries it. **Permanent and justified by scope**: revisit if the real E-Links API is confirmed to leave out `name`. | Following the Swagger exactly (no `name`) was offered and declined by the user. |
| **Deferral**: the Sonar step in CI stays inactive until SonarQube Cloud is reconfigured (Principle XIII, Governance) | The existing SonarQube Cloud project (`hmcts_ctam-jomockapi2`) appears to use **Automatic Analysis**, which can't run alongside a CI `./gradlew sonar` run and doesn't import JaCoCo coverage. Switching it off and adding the `SONAR_TOKEN` secret needs a SonarQube Cloud project admin, which this feature can't do. Until then, `ci.yml` guards the step with `if: env.SONAR_TOKEN != ''` so CI stays green. Sonar analysis keeps running through Automatic Analysis, but outside the build and without coverage. That falls short of "MUST run as part of the build" and counts as a skipped gate. **Way back to compliance**: (1) before this feature's PR merges, raise an issue asking a SonarQube Cloud admin to switch Automatic Analysis off (Administration → Analysis Method) and add `SONAR_TOKEN`; (2) once the secret exists, remove the `if:` guard so the Sonar step is unconditional and a failing quality gate fails CI. The deferral is closed by the PR that removes the guard, which must be the next change to `ci.yml` after the secret is added. | Running `./gradlew sonar` unconditionally now would fail every CI run, because there is no token and Automatic Analysis conflicts with it. That would block all merges on a setting this repository can't change. Leaving Sonar out of CI entirely would give up on build-integrated analysis altogether. |
| *Conditional*: CodeQL may not support Java 25 (Principle XIII) | Only recorded if the Setup check in research R11 (risk 2) fails. | If it fails: CodeQL is configured and kept, but marked non-blocking until upstream supports Java 25. **Way back to compliance**: make it blocking in the first CodeQL release that supports Java 25, tracked by an issue raised at that point. |

## Post-Design Constitution Re-Check

This was re-run after Phase 1 (research.md, data-model.md, contracts/reference-data-api.md, quickstart.md).

The design brought in no dependency, layer or abstraction beyond what the initial check assessed.

Three design details strengthen the assessments:
- The contract fixes every error message, which supports V and IX.
- The OpenAPI `attribute_name` enum is generated from the registry, which supports VII and XVIII.
- Checks run in a fixed order (authentication before routing), which supports XV.

The data model's startup validation makes FR-022 impossible to violate at runtime rather than something tests have to catch. Every row stays **Pass**. Two items remain open: the documented Sonar CI deferral (Complexity Tracking) and the conditional CodeQL/Java 25 risk.
