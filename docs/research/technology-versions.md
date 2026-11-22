# Research: current Technology versions and official compatibility

Ticket: joshaustintech/prefab#2 (child of map #1). Checked on **2026-11-21** against primary sources: official sites, support policies, docs, source code and release tags. Feeds how the **Compatibility Matrix** is declared.

## TL;DR

| Role | Technology | Current release (2026-11-21) | Status of the v1 target |
|---|---|---|---|
| language | Java | 25 LTS (GA 2025-09-16); 27 non-LTS is the newest GA | Target **Java 25** exists. It is the current LTS. |
| framework | Spring Boot | **4.2.0** (2026-11-19) | Target **4.2** went GA two days ago. |
| language | Ruby | **4.0.10** (4.0 branch released 2025-12-25) | Target **Ruby 4** exists. |
| framework | Rails | **8.1.4.1** (2026-11-04) | **Rails 8.2 is NOT released.** `main` is `8.2.0.alpha`. There is no beta, RC, or `8-2-stable` branch. |
| datastore | PostgreSQL | **19.0** (2026-11-19); 15–18 still supported | |
| datastore | SQLite | **3.54.1** (2026-10-16) | |
| messaging | Apache Kafka | **4.4.0** (2026-10-22); supported lines 4.4, 4.3, 4.2 | |
| messaging client (Spring) | Spring for Apache Kafka | **4.2.0** (2026-11-19), BOM `kafka-clients` 4.4.0 | |
| messaging client (Rails) | Karafka (+ WaterDrop, karafka-rdkafka) | Karafka 2.6.1, WaterDrop 2.10.11, karafka-rdkafka 0.32.2 | |

Implications for the Compatibility Matrix:

- **Rails 8.2 can't be a shipped Technology version today.** Either use Rails 8.1 for v1, or enter 8.2 as a prerelease or provisional row with its declared constraints (Ruby >= 3.3.5) and confirm it when 8.2 ships.
- **Spring Boot's Java support is a range** (min 17, max 27 for 4.0, 4.1 and 4.2), so the matrix needs min/max ranges, not single pairs. Rails declares only a minimum (`required_ruby_version`). Rails CI defines the upper bound that is actually tested.
- **Datastore minimums are framework-dependent,** for example Hibernate 7.4 Postgres >= 14 and Active Record SQLite >= 3.23.0. The matrix can record framework × datastore constraints, not just language × framework.
- **SQLite with Spring Boot works, but only community support covers it.** The details are below.

## Java

Source: OpenJDK project page (https://openjdk.org/projects/jdk/) and Oracle Java SE Support Roadmap (https://www.oracle.com/java/technologies/java-se-support-roadmap.html).

- GA dates: JDK 25 2025-09-16, JDK 26 2026-03-17, JDK 27 2026-09-15. JDK 28 is in development (March 2027).
- Oracle marks 21 and 25 as LTS and 26, 27, 28 as non-LTS. The roadmap says the "next planned LTS release is Java 29 in September 2027".
- Oracle Premier Support for Java 25 runs to September 2030, and Extended Support to September 2033.

## Spring Boot and Java

Sources: Spring Boot system requirements pages (https://docs.spring.io/spring-boot/system-requirements.html, `/4.1/…`, `/4.0/…`), the Spring generations API (https://api.spring.io/projects/spring-boot/generations, which backs https://spring.io/projects/spring-boot#support), GitHub release tags (`spring-projects/spring-boot`), and the support policy (https://github.com/spring-projects/spring-boot/wiki/Supported-Versions).

| Spring Boot | Latest patch | Java (official) | Spring Framework | Initial release | OSS support ends | Commercial ends |
|---|---|---|---|---|---|---|
| 4.2.x | 4.2.0 (2026-11-19) | **min 17, "compatible with versions up to and including Java 27"** | 7.1.0+ | 2026-11 | 2027-12-31 | 2028-12-31 |
| 4.1.x | 4.1.3 (2026-11-19) | min 17, up to and including 27 | 7.0.11+ | 2026-06 | 2027-07-31 | 2028-07-31 |
| 4.0.x | 4.0.10 (2026-11-19) | min 17, up to and including 27 | 7.0.11+ | 2025-11 | 2026-12-31 | 2027-12-31 |
| 3.5.x | — | (min 17) | 6.2.x | 2025-05 | 2026-06-30 (ended) | 2032-06-30 |

Quotes from the 4.2.0 docs: "Spring Boot 4.2.0 requires at least Java 17 and is compatible with versions up to and including Java 27." "Spring Framework 7.1.0 or above is also required."

Other 4.2.0 requirements: Maven 3.6.3+; Gradle 8.14+ or 9.x; Tomcat 11.0.x or Jetty 12.1.x (Servlet 6.1); GraalVM 25+ for native images.

Policy: a new minor ships every six months (May and November). Each minor gets at least 12 months of OSS support, and a major gets at least 3 years.

The 4.2 milestone history shows the cadence: M1 2026-08-20, M2 2026-09-24, RC1 2026-10-22, RC2 2026-11-05, GA 2026-11-19.

Versions that Spring Boot 4.2.0 manages (https://docs.spring.io/spring-boot/appendix/dependency-versions/coordinates.html):

| Artifact | Version |
|---|---|
| `org.springframework.kafka:spring-kafka` | 4.2.0 |
| `org.apache.kafka:kafka-clients` | 4.4.0 |
| `org.hibernate.orm:hibernate-core` | 7.4.15.Final |
| `org.hibernate.orm:hibernate-community-dialects` | 7.4.15.Final |
| `org.postgresql:postgresql` (pgJDBC) | 42.7.14 |
| `org.xerial:sqlite-jdbc` | 3.53.4.0 |
| `org.flywaydb:flyway-core` / `flyway-database-postgresql` | 12.11.0 |
| `org.liquibase:liquibase-core` | 5.0.4 |
| `org.testcontainers:testcontainers` | 2.0.5 |
| Spring Modulith | not managed by the Boot BOM; it has its own BOM |

## Ruby

Sources: https://www.ruby-lang.org/en/downloads/ and https://www.ruby-lang.org/en/downloads/branches/

| Branch | Latest | Status | Released | EOL |
|---|---|---|---|---|
| 4.1 | 4.1.0-preview1 | preview | (Ruby ships a minor each Dec 25) | — |
| **4.0** | **4.0.10** | normal maintenance | 2025-12-25 | TBD |
| 3.4 | 3.4.13 | normal maintenance | 2024-12-25 | TBD |
| 3.3 | 3.3.14 | security maintenance | 2023-12-25 | 2027-03-31 (expected) |
| 3.2 | 3.2.11 | EOL | 2022-12-25 | 2026-04-01 |

Ruby 4.0 changes that affect generated apps: Bundler 4 ships with Ruby 4.0, and more default gems become bundled gems (for example `ostruct`, `benchmark`, `logger`; `cgi` is removed apart from `cgi/escape`). Apps on Rails 8.1.1 hit some gem load errors (rails/rails#56457, closed 2025-12-29). These come from community reports and the issue tracker, not from official release notes.

## Rails and Ruby

Sources: https://rubyonrails.org/ (latest release banner), https://rubyonrails.org/maintenance, `rails/rails` tags and branches through the GitHub API, `rails.gemspec` on `main` and `8-1-stable`, `rails/buildkite-config` (the CI Ruby matrix), and https://edgeguides.rubyonrails.org/8_2_release_notes.html

| Rails | Latest | Ruby (gemspec minimum) | Ruby tested in official CI | Bug fixes until | Security fixes until |
|---|---|---|---|---|---|
| **8.2** | **unreleased** (`RAILS_VERSION` on `main` = `8.2.0.alpha`; no `v8.2.*` tag; no `8-2-stable`) | `>= 3.3.5` | 3.3, 3.4, 4.0 | — | — |
| 8.1 | 8.1.4.1 (2026-11-04) | `>= 3.2.0` | 3.2, 3.3, 3.4, 4.0 | 2026-10-10* | 2027-10-10 |
| 8.0 | 8.0.5.2 | `>= 3.2.0` | 3.2–4.0 | ended | 2026-11-07 (ended) |

\* The maintenance policy says that if no new feature release ships within a year, support for the previous release is extended until the next one ships. 8.2 hasn't shipped, so 8.1 is in effect still the supported feature line.

How the CI tested range is derived: `rails/buildkite-config` runs the Ruby list `2.4 … 3.4 4.0`. It filters out versions below the gemspec's `required_ruby_version`, and only marks Rubies above a cap as soft-fail for Rails < 8. So for Rails >= 8, Ruby 4.0 is a hard-fail (fully supported) CI target. No official Rails page publishes a "max Ruby", so the Ruby upper bound has to come from CI, not from a declared policy.

What the 8.2 edge release notes (work in progress) say: they remove the Sidekiq Active Job adapter (it moves into the `sidekiq` gem) and deprecate several built-in Active Job adapters. They give no minimum database versions.

Active Record datastore minimums (8-1-stable source):

- PostgreSQL adapter: "Active Record supports PostgreSQL >= 9.5". The `pg` gem is at 1.7.0.
- SQLite3 adapter: "Active Record supports SQLite >= 3.23.0". It requires `gem "sqlite3", ">= 2.1"`, and the latest is 2.10.2.

## PostgreSQL

Source: https://www.postgresql.org/support/versioning/ and the news archive ("PostgreSQL 19 Released!", 2026-11-19).

| Major | Current minor | Released | EOL |
|---|---|---|---|
| **19** | 19.0 | 2026-11-19 | 2031-11-19 |
| 18 | 18.7 | 2025-09-25 | 2030-11-14 |
| 17 | 17.12 | 2024-09-26 | 2029-11-08 |
| 16 | 16.16 | 2023-09-14 | 2028-11-09 |
| 15 | 15.20 | 2022-10-13 | 2027-11-11 |

Each major is supported for 5 years. Hibernate 7.4 (core `PostgreSQLDialect`) needs PostgreSQL **>= 14.0** (https://docs.hibernate.org/stable/orm/dialect/). Active Record needs >= 9.5. pgJDBC is 42.7.14.

Postgres 19 shipped in November, later than the usual September slot. The news archive gives no reason. Its ecosystem support (Docker images, Testcontainers, Flyway "verified" lists) may lag by a few weeks.

## SQLite

Source: https://sqlite.org/ — latest **3.54.1** (2026-10-16). It has no "support lines". SQLite has a single maintained 3.x line.

- Java driver: `org.xerial:sqlite-jdbc` 3.54.1.0 (2026-10-29, GitHub releases). Spring Boot 4.2.0 manages 3.53.4.0.
- Ruby driver: the `sqlite3` gem 2.10.2. It bundles its own SQLite by default.

## Kafka

Source: https://kafka.apache.org/community/downloads/ and the compatibility page (https://kafka.apache.org/44/getting-started/compatibility/).

- Latest: **4.4.0** (2026-10-22). The "Supported releases" are 4.4.0, 4.3.2 and 4.2.2. The project aims for three releases a year, and bugfix releases ship only for supported lines. No explicit EOL dates are given.
- Kafka 4.x is KRaft only (ZooKeeper was removed in 4.0). Java: clients and Streams run on Java 11+, and brokers and Connect need Java 17+. The compatibility page still lists only the "Java 11, 17, 23" figures from 4.0. Clients are fully compatible with 3.x brokers, partially with 2.1–2.8, and not at all with brokers below 2.1.

### Spring ecosystem: Spring for Apache Kafka

Sources: the GitHub releases for `spring-projects/spring-kafka` and https://api.spring.io/projects/spring-kafka/generations

| spring-kafka | Paired Spring Boot | Latest | OSS ends | Commercial ends |
|---|---|---|---|---|
| 4.2.x | 4.2.x | 4.2.0 (2026-11-19) | 2027-12-31 | 2028-12-31 |
| 4.1.x | 4.1.x | 4.1.3 | 2027-07-31 | 2028-07-31 |
| 4.0.x | 4.0.x | 4.0.9 | 2026-12-31 | 2027-12-31 |

Its version follows the Boot line one-to-one. For a generated Service, the idiomatic choice is to take both `spring-kafka` and `kafka-clients` from the Boot BOM, with no explicit versions.

### Rails ecosystem: Karafka

Sources: https://karafka.io/docs/Upgrades-Versions-Lifecycle-and-EOL/, the `karafka/karafka` gemspec, and RubyGems.

- Rails itself has no Kafka integration. The idiomatic, actively maintained option is **Karafka** (consumers) plus **WaterDrop** (its producer). Both run on **`karafka-rdkafka`**, Karafka's fork of the librdkafka bindings, rather than the upstream `rdkafka` gem (0.31.0). `karafka` 2.6.1 depends on `karafka-rdkafka >= 0.32.2` and `waterdrop >= 2.10.2, < 3`.
- Karafka requires Ruby `>= 3.3.0` (gemspec). Its supported Ruby versions are 4.0 (to 2029-09-30), 3.4 and 3.3. It supports Rails 8.1 (active), 8.0 (to 2027-05-31) and 7.2 (to 2027-02-28), and "requires Rails 7.2 or newer".
- Karafka 2.6 is active and 2.5 is in maintenance until 2026-12-31. It supports every non-EOL Kafka version (librdkafka works with brokers >= 1.0).
- Some features (for example parts of the Web UI and advanced patterns) are Karafka **Pro** (commercial). The OSS core covers plain produce and consume.

## SQLite with Spring Boot: viability and caveats

It works, but no first-party Spring or Hibernate support covers it.

What works:

- **Driver:** Boot's `DatabaseDriver` enum has `SQLITE("SQLite", "org.sqlite.JDBC")` (v4.2.0 source, `module/spring-boot-jdbc/.../DatabaseDriver.java`). The driver class is inferred from a `jdbc:sqlite:` URL, and the BOM manages `sqlite-jdbc`.
- **Dialect:** `org.hibernate.community.dialect.SQLiteDialect` ships in `hibernate-community-dialects` (version managed by the Boot BOM). Hibernate's docs say community dialects "are maintained on a best-effort basis by vendors or individuals". They aren't tested by the Hibernate team, and the dialect table lists SQLite's minimum version as "N/A". You must set `spring.jpa.database-platform` (or `hibernate.dialect`) explicitly.
- **Migrations:** Flyway supports SQLite in `flyway-core` (`org.flywaydb.core.internal.database.sqlite`; no separate `flyway-database-sqlite` artifact exists). Redgate lists SQLite 3.7+.

Caveats:

1. **Not an "embedded database" for Boot.** `EmbeddedDatabaseConnection` covers only H2, Derby (deprecated) and HSQLDB. As a result, no in-memory auto-configuration happens, and `spring.sql.init` / `ddl-auto` defaults treat SQLite as a regular database. Generated config must set the URL, dialect and init behaviour explicitly.
2. **Locking.** The `SQLiteDialect` source returns no `FOR UPDATE` clause ("SQLite does not support the FOR UPDATE clause"), so JPA pessimistic locks silently become no-ops. Flyway likewise can't run concurrent migrations ("SQLite lacks `SELECT ... FOR UPDATE`").
3. **Concurrency.** SQLite has a single writer per database file. With a pool (HikariCP), writes contend, so keep the pool small (or at 1 for writes) and enable WAL mode. This is a general SQLite property and an inference, not a statement from Spring docs.
4. **Flyway limits:** no multiple schemas, and no `BEGIN`/`COMMIT` inside migrations (SQLite has no nested transactions). Some statements must run as non-transactional migrations.
5. **Support level:** it is community-maintained (dialect) and has no Testcontainers story (SQLite is a file, which also makes it easy to test). Hibernate's core-tested dialects don't include SQLite.

Rails, by contrast, treats SQLite as a first-class production datastore (the SQLite3 adapter is in core, minimum SQLite 3.23.0). So "SQLite" in the Compatibility Matrix means a different support level per framework.

## Open questions or surprises

- **Rails 8.2 doesn't exist as a release** (only `8.2.0.alpha` on `main`). The map's v1 target needs a decision: target 8.1, or allow prerelease Technology versions in the matrix.
- **Spring Boot 4.2.0 is 2 days old.** Its first patch, 4.2.1, is in snapshot.
- **PostgreSQL 19 is also 2 days old**, and its tooling may lag. Pick which Postgres majors the matrix lists (for example 15–19, or 17–19).
- **Hibernate 8.0 is at CR1** (tags), so a future Boot line will likely move to it. It has no effect on 4.2.
- **The Ruby upper bound comes from CI, not from a policy.** The matrix must say whether "compatible" means a declared minimum or the CI-tested range.
