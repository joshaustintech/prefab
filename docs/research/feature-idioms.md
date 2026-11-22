# Feature idioms: logging, tracing, unit tests, smoke tests in Spring Boot 4 and Rails 8

Research for issue #4 (child of map #1). Researched 2026-11-21 against primary sources.
Vocabulary follows `GLOSSARY.md` (Feature, Stack, Technology, Compatibility Matrix).

## Version check (as of 2026-11-21)

| Technology | Current | Source |
|---|---|---|
| Spring Boot | **4.2.0** is the current stable reference docs version (4.2.1-SNAPSHOT in progress) | [Spring Boot tracing docs](https://docs.spring.io/spring-boot/reference/actuator/tracing.html) |
| JUnit | **6.1.3** (Platform, Jupiter and Vintage share one version; Java 17+ at runtime; Vintage deprecated) | [JUnit User Guide](https://docs.junit.org/current/user-guide/) |
| Testcontainers (Java) | 2.0.x (the Boot 4.2 docs link 2.0.5 javadoc; artifacts are renamed `testcontainers-<module>`, e.g. `testcontainers-postgresql`, `testcontainers-kafka`) | [Boot Testcontainers docs](https://docs.spring.io/spring-boot/reference/testing/testcontainers.html) |
| Rails | **8.1.4.1** is the latest release (2026-11-04). **Rails 8.2 is not released**: `main` reports `8.2.0.alpha`, and there is no beta or RC post | [Rails releases](https://rubyonrails.org/category/releases), [RAILS_VERSION on main](https://github.com/rails/rails/blob/main/RAILS_VERSION) |
| Ruby | **4.0.10** (2026-11-18); 3.4.13 is the same day | [Ruby releases](https://www.ruby-lang.org/en/downloads/releases/) |
| OpenTelemetry Ruby | Traces **Stable**, Metrics **Development**, Logs **Development** | [OTel Ruby](https://opentelemetry.io/docs/languages/ruby/) |

**Impact on the map:** the map names "Rails 8.2" as a v1 Technology, and it does not exist yet. The Compatibility Matrix should ship with Rails 8.1 and add 8.2 as data when it is released. Everything below is checked against 8.1 and the `main` generator templates.

## Per Feature × ecosystem

### Logging

| | Spring Boot 4.2 / Java 25 | Rails 8.1 / Ruby 4 |
|---|---|---|
| API in app code | SLF4J (`LoggerFactory.getLogger`). Boot routes JUL, Commons Logging, Log4j and SLF4J into the backend. Key/value pairs use the SLF4J fluent API `addKeyValue` | `Rails.logger` / `logger.info { ... }` (pass a block so the string is only built when the level is on) |
| Default backend | Logback, via the starters | `ActiveSupport::Logger`, wrapped in `ActiveSupport::TaggedLogging` |
| Structured output | Built in. `logging.structured.format.console=ecs` (or `gelf`, `logstash`, or the FQCN of a custom `StructuredLogFormatter`). `logging.structured.format.file` for files. MDC entries go into the JSON. ECS service fields default from `spring.application.name` / `spring.application.version` | **Rails 8.1 Structured Event Reporting**: `Rails.event.notify("name", payload)`, `Rails.event.tagged`, `Rails.event.set_context` (reset per request or job), `Rails.event.subscribe(obj)` where `obj#emit(event)` does the encoding (e.g. JSON). Plain `Rails.logger` lines are still text with tags. No JSON log formatter is built in |
| Production default | Console, plain pattern unless structured format is set | Generated `production.rb`: `config.logger = ActiveSupport::TaggedLogging.logger(STDOUT)`, `config.log_tags = [ :request_id ]`, `config.log_level = ENV.fetch("RAILS_LOG_LEVEL", "info")`, `config.silence_healthcheck_path = "/up"` |
| Trace correlation | Automatic: Boot adds `[traceId-spanId]` from the MDC when tracing is on. Rename keys with `management.tracing.mdc.trace-id-key`/`span-id-key`. Change the pattern with `logging.pattern.correlation` | Not built in. Add a `log_tags` lambda that reads `OpenTelemetry::Trace.current_span.context`, or add trace IDs in an event-reporter subscriber (this is Prefab's convention to define; it is not a Rails default) |

Sources: [Boot logging](https://docs.spring.io/spring-boot/reference/features/logging.html), [Boot tracing (log correlation)](https://docs.spring.io/spring-boot/reference/actuator/tracing.html), [Rails debugging guide](https://guides.rubyonrails.org/debugging_rails_applications.html), [Rails 8.1 release notes](https://guides.rubyonrails.org/8_1_release_notes.html), [ActiveSupport::EventReporter API](https://edgeapi.rubyonrails.org/classes/ActiveSupport/EventReporter.html), [Rails production.rb template](https://github.com/rails/rails/blob/main/railties/lib/rails/generators/rails/app/templates/config/environments/production.rb.tt), [Rails configuring guide](https://guides.rubyonrails.org/configuring.html).

### Tracing

| | Spring Boot 4.2 | Rails 8.1 |
|---|---|---|
| Idiom | Micrometer Observation as the API (`ObservationRegistry`, `Observation...observe()`; `@Observed` needs `management.observations.annotations.enabled=true` and `spring-boot-starter-aspectj`). Micrometer Tracing is the facade. The OTel bridge exports | OpenTelemetry Ruby SDK plus auto-instrumentation. Rails has no tracing of its own |
| Dependency | `org.springframework.boot:spring-boot-starter-opentelemetry` (new in Boot 4). The alternative is `spring-boot-starter-zipkin` (Brave) | `bundle add opentelemetry-sdk opentelemetry-instrumentation-all opentelemetry-exporter-otlp` |
| Wiring | Auto-configured. `management.opentelemetry.tracing.export.otlp.endpoint`, `management.opentelemetry.resource-attributes.*` | `config/initializers/opentelemetry.rb`: `OpenTelemetry::SDK.configure { \|c\| c.service_name = ...; c.use_all }` |
| Service name / resource | Merges `OTEL_SERVICE_NAME` and `OTEL_RESOURCE_ATTRIBUTES`. The Boot property wins on conflict | Standard `OTEL_*` env vars, or `c.service_name` in code |
| Endpoint | Property above. Default is none, so it must be set | `OTEL_EXPORTER_OTLP_ENDPOINT`. Default is `localhost:4318` (OTLP/HTTP) |
| Sampling | **Default 10%** (`management.tracing.sampling.probability`). Sampler `management.opentelemetry.tracing.sampler`, default `parent-based-trace-id-ratio` | OTel spec default `parentbased_always_on` (`OTEL_TRACES_SAMPLER`), i.e. 100% |
| Propagation | W3C via the auto-configured `RestClient.Builder` / `WebClient.Builder` only | OTel default `tracecontext,baggage` |
| Logs via OTLP | Boot 4 has `management.opentelemetry.logging.export.otlp.endpoint` and an `SdkLoggerProvider`, but **does not bridge Logback to it out of the box**. A third-party appender is needed | OTel Ruby logs are in Development. Skip OTLP log export in v1 |
| In tests | Reporting tracing components are **not** auto-configured in `@SpringBootTest` | (n/a; exporters are controlled by env) |

The OpenTelemetry project's own Java agent and its `opentelemetry-spring-boot-starter` are alternatives. OTel calls the agent "the default choice" for Spring Boot. For generated code, Boot's first-party starter is the idiomatic choice: no agent, config lives in `application.properties`, and it is version-locked to the Boot BOM. Picking Boot's starter is my inference, not a statement in either set of docs.

Sources: [Boot tracing](https://docs.spring.io/spring-boot/reference/actuator/tracing.html), [Boot observability](https://docs.spring.io/spring-boot/reference/actuator/observability.html), [Boot application properties](https://docs.spring.io/spring-boot/appendix/application-properties/index.html), [OTel Ruby getting started](https://opentelemetry.io/docs/languages/ruby/getting-started/), [OTel Ruby exporters](https://opentelemetry.io/docs/languages/ruby/exporters/), [OTel SDK env vars](https://opentelemetry.io/docs/languages/sdk-configuration/general/), [OTel Spring Boot starter](https://opentelemetry.io/docs/zero-code/java/spring-boot-starter/).

### Unit tests

| | Spring Boot 4.2 | Rails 8.1 |
|---|---|---|
| Framework | JUnit 6 (Jupiter) through `spring-boot-starter-test`, which brings JUnit, Spring Test, AssertJ, Hamcrest, Mockito, JSONassert, JsonPath, Awaitility | **Minitest**, the Rails default. The Rails guides do not mention RSpec. RSpec is popular, but a generator that matches `rails new` should use Minitest |
| Plain unit | POJO plus JUnit, no Spring context | `ActiveSupport::TestCase` in `test/models` etc., with YAML fixtures in `test/fixtures` and a transaction rollback per test |
| Slices / layers | Boot 4 splits slices into per-technology `-test` modules: `@WebMvcTest` in `spring-boot-webmvc-test` (package `org.springframework.boot.webmvc.test.autoconfigure`), `@DataJpaTest` in `spring-boot-data-jpa-test`, `@JdbcTest` in `spring-boot-jdbc-test`, `@RestClientTest` in `spring-boot-restclient-test`, plus `spring-boot-micrometer-tracing-test`. No Kafka slice module is listed | Controller and request tests: `ActionDispatch::IntegrationTest`. Also `ActiveJob::TestCase`, `ActionMailer::TestCase`. System tests (`ActionDispatch::SystemTestCase`, Capybara/Selenium) are no longer generated by scaffolds. They are not relevant for API Services |
| HTTP assertions | `MockMvcTester` (AssertJ) via `@AutoConfigureMockMvc` / `@WebMvcTest`. `RestTestClient` via `@AutoConfigureRestTestClient` (new in Boot 4) | `get "/path"; assert_response :success` |
| Runner | `./mvnw test` / `./gradlew test` | `bin/rails test`. Rails 8.1 also adds local CI (`config/ci.rb`, run by `bin/ci`) |

Sources: [Boot testing](https://docs.spring.io/spring-boot/reference/testing/index.html), [Boot test modules](https://docs.spring.io/spring-boot/reference/testing/test-modules.html), [Boot test scope dependencies](https://docs.spring.io/spring-boot/reference/testing/test-scope-dependencies.html), [Testing Spring Boot applications](https://docs.spring.io/spring-boot/reference/testing/spring-boot-applications.html), [JUnit User Guide](https://docs.junit.org/current/user-guide/), [Rails testing guide](https://guides.rubyonrails.org/testing.html), [Rails 8.1 release notes](https://guides.rubyonrails.org/8_1_release_notes.html).

### Smoke tests

Neither ecosystem has a named "smoke test" construct. The conventional shape in both is to **boot the real app against real backing services and hit the health endpoint, plus one happy-path request per Component**.

| | Spring Boot 4.2 | Rails 8.1 |
|---|---|---|
| Health endpoint | Actuator `/actuator/health` (the only endpoint exposed over HTTP by default). Liveness and readiness groups at `/actuator/health/liveness` and `/readiness`, on by default. With `management.endpoint.health.probes.add-additional-paths=true` they are also served at `/livez` and `/readyz` on the main port. `DOWN`/`OUT_OF_SERVICE` return 503 | `get "up" => "rails/health#show"` in the generated routes. Returns 200 if the app boots without exceptions, otherwise 500. **It does not check the DB or Kafka** |
| In-process smoke test | `@SpringBootTest(webEnvironment = RANDOM_PORT)` + `@AutoConfigureRestTestClient`, GET the health endpoint, assert 200 | `ActionDispatch::IntegrationTest`: `get rails_health_check_path; assert_response :success` |
| Real dependencies | Testcontainers: `@Testcontainers` + `@Container @ServiceConnection static PostgreSQLContainer ...` (needs `spring-boot-testcontainers`). Service connections exist for `KafkaContainer`, `ConfluentKafkaContainer`, `RedpandaContainer`. Boot recommends container `@Bean`s in a `@TestConfiguration` (or `@ImportTestcontainers`) over static fields when contexts are cached | No first-party Testcontainers integration. The idiom is a real DB from `config/database.yml` (SQLite by default; Postgres via `DATABASE_URL` / CI service) |
| Out-of-process | Run the built jar or container and curl `/actuator/health/readiness` | Run the container (Rails 8 generates a `Dockerfile`) and curl `/up` |

Sources: [Boot endpoints / health](https://docs.spring.io/spring-boot/reference/actuator/endpoints.html), [Boot Testcontainers](https://docs.spring.io/spring-boot/reference/testing/testcontainers.html), [Testing Spring Boot applications](https://docs.spring.io/spring-boot/reference/testing/spring-boot-applications.html), [Rails routes template](https://github.com/rails/rails/blob/main/railties/lib/rails/generators/rails/app/templates/config/routes.rb.tt).

## Shared conventions (put these in the Universal Description Spec)

1. **OpenTelemetry/OTLP as the tracing contract.** Both export OTLP/HTTP and honour `OTEL_SERVICE_NAME` / `OTEL_RESOURCE_ATTRIBUTES`. The spec's tracing Feature can be expressed as {service name, OTLP endpoint, sample ratio}, and each template maps it. Set the sample ratio **explicitly**, because the defaults differ (Boot 10%, OTel Ruby 100%).
2. **W3C `traceparent` propagation** is the default on both sides, so traces cross Spring and Rails Services in one System.
3. **A health endpoint returning 200/5xx** exists on both, which gives the smoke-test Feature a uniform probe. The paths differ (`/actuator/health[/readiness]` vs `/up`). Prefab could normalise them: Boot's `add-additional-paths` gives `/livez` and `/readyz`, or Rails could add a route. This is a design decision for later.
4. **Logs to stdout with a request/trace id** is the production idiom on both.
5. **Smoke test = boot + health + one call**, run against real backing services (compose or containers).

## Divergences (keep these in per-Technology templates)

- **Structured logging:** Boot has JSON formatters built in (ECS/GELF/Logstash). Rails 8.1 has a structured *event* API (`Rails.event`) whose subscribers encode the events, but `Rails.logger` output stays text. A JSON log Feature in Rails needs a generated subscriber or formatter.
- **Trace↔log correlation:** automatic in Boot. In Rails it has to be generated (a `log_tags` lambda or subscriber).
- **OTLP logs:** neither works first-party. Boot needs a third-party Logback bridge and OTel Ruby logs are in Development. Recommend logs to stdout only in v1.
- **Test framework and layering:** JUnit 6 with per-technology slice modules (Boot 4 split them, so templates must add `spring-boot-<tech>-test` deps matching each Component's Technology) vs Minitest with fixtures and integration tests.
- **Real dependencies in tests:** Testcontainers with `@ServiceConnection` is first-class in Boot. Rails uses configured databases. Testcontainers-Ruby is not a Rails idiom.
- **Health depth:** Actuator health aggregates DB/Kafka indicators. Rails `/up` only proves boot.

## Open questions surfaced

- Rails 8.2 is unreleased. Should v1's Compatibility Matrix target Rails 8.1 (with Ruby 3.4/4.0) and add 8.2 when it ships?
- Should Prefab normalise health paths across Stacks (e.g. `/livez` / `/readyz`), or accept each ecosystem's native path?
- Should the Rails smoke-test Feature generate a deeper health check (DB/Kafka) so it matches Actuator?
- Structured logging in Rails: generate an `ActiveSupport::EventReporter` subscriber plus a JSON log formatter, or keep text logs with tags?
- Smoke tests across a System need the "local docker-compose" item already listed under "Not yet specified" on the map.
