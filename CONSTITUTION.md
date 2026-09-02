# Log-Control Starter — Project Constitution

**Status:** Draft v1 — approved decisions from project kickoff (2026-08-31); dual Spring Boot generation support (3.5.x + 4.1.1) reaffirmed 2026-09-01 as v1 scope, both needed now and released together (see §2.9, §2.12)

## 1. Mission

Build a Spring Boot starter library that any REST service in the organization can depend on to get consistent, root-cause-friendly logging with zero-to-minimal integration effort. Every log line written through this library must let a person reading it identify, at a glance, *what request was being handled* and *what specific point in the business logic* produced the line — without needing to trace back through code or reconstruct context by hand.

This document is the constitution for the project: the set of decisions that are **not up for casual debate** during implementation. Anything not listed here is an open implementation detail to be resolved as the design proceeds.

## 2. Non-Negotiables

### 2.1 Core guarantee
Every log line emitted through this library's API automatically carries three pieces of context, in addition to the developer's message text:

1. **HTTP method** of the request being handled (GET, POST, PUT, DELETE, etc.).
2. **Endpoint** being served — the matched route template (e.g. `/orders/{id}`) is the primary value, since it groups identical endpoints together for log search/aggregation regardless of the actual path variables passed. The raw resolved path may be captured as a secondary field, and is the only option available when no route match exists yet (e.g. a 404 before dispatch).
3. **Situation code** — a unique identifier for the specific point in the logic flow where the log call happened, sufficient on its own to jump straight to the responsible code path.

These three fields must appear on **every** log line written via the library's API. There is no "plain" logging mode that skips them.

### 2.2 Situation codes are a typed catalog, not free strings
- Each consuming service defines its own catalog of situation codes as an **enum implementing a shared interface** exposed by the library (e.g. a `LogSituationCode` contract with a code value and a human-readable description).
- Magic/free-form strings are not a supported way to supply a code. This guarantees uniqueness within a service, compile-time discoverability (IDE autocomplete over the enum), and a browsable catalog per service.
- The library ships one built-in fallback code (e.g. `UNSPECIFIED`) reserved for the case below — it is never a substitute for real per-call codes and should be treated as a signal to fix the call site.

### 2.3 Code is mandatory, with a guarded fallback
- The primary logging API requires a situation code argument — there is no overload that silently omits it.
- If a call somehow reaches the logging layer without a resolvable code (e.g. during incremental migration of legacy code), the library substitutes its built-in fallback code rather than dropping the field or throwing. The field is **never absent** from a log line; at worst it is `UNSPECIFIED`.
- Emitting the fallback code should be visible/noisy enough (e.g. a startup warning, static-analysis-friendly usage, or a metrics counter) that it doesn't become the permanent state for a call site.

### 2.4 Dual stack support: Spring MVC and WebFlux
- The library supports both servlet-based (`spring-boot-starter-web`) and reactive (`spring-boot-starter-webflux`) consuming services, with equivalent guarantees on both.
- Context capture uses framework-idiomatic interception points: a `HandlerInterceptor`/`OncePerRequestFilter` for MVC, a `WebFilter` for WebFlux.
- Because MVC's MDC (`ThreadLocal`) approach does not carry across reactive execution, WebFlux context must be propagated through Reactor Context (bridged to MDC at the point of writing each log line, e.g. via Micrometer's context-propagation library) rather than assumed to behave like a thread-local.
- Auto-configuration must detect which stack is present on the consuming service's classpath and wire up only the relevant integration (no requirement for a service to depend on both).

### 2.5 Configurable dual output format
- Both a structured JSON format (for log aggregators — ELK/Datadog/Splunk-style ingestion) and a human-readable text pattern (for local/console use) are first-class, supported outputs.
- Switching between them is a configuration change (e.g. Spring profile or `application.yml` property), never a code change in the consuming service.

### 2.6 Logback only
- The library targets Logback as its logging backend exclusively (Spring Boot's default). It does not need to remain backend-agnostic via bare SLF4J, and is not required to support Log4j2.
- Both output formats (2.5) are delivered as Logback encoders/appenders shipped by the library.

### 2.7 No sensitive data leakage by default
- The library must not, by default, log request/response bodies, `Authorization`/cookie headers, or other credential-bearing values as part of its automatic context enrichment.
- Any future capability to capture request/response payloads must be explicitly opt-in and documented as a security-sensitive setting.

### 2.8 Spring Boot starter conventions
- The library follows standard Spring Boot starter conventions: auto-configuration classes registered via `AutoConfiguration.imports`, conditional beans (`@ConditionalOnClass`, `@ConditionalOnMissingBean`, etc.), and externalized configuration via a `@ConfigurationProperties` prefix.
- Adopting the library in a consuming service should require, at minimum: adding the Maven dependency, defining that service's situation-code enum, and (optionally) a few configuration properties. No manual bean wiring should be required for the default behavior.

### 2.9 Technical baseline
- **Build tool:** Maven.
- **Language / runtime baseline:** Java 17+.
- **Spring Boot baseline:** dual-generation support, reaffirmed 2026-09-01 (see §2.12): the
  organization runs a MIX of services today — some already on **Spring Boot 4.1.1** in production,
  others still on **Spring Boot 3.5.x** with a migration to 4.x planned but not yet complete. Both
  generations need this library released together as v1 (confirmed 2026-09-01); coding order is
  sequenced (3.5.x starter first — see §2.12) but that is an implementation-sequence choice only
  and does not delay or split the release. 3.5.x is the floor for the 3.x line (past its own OSS
  end-of-life as of 2026-06-30, but still running production services; 3.2 through 3.4 are also no
  longer OSS-supported and are not targeted); Spring Boot 4.1.x is the floor for the 4.x line,
  pinned to the exact minor (4.1.1) confirmed running in production as of 2026-09-01. See §2.12
  for why and how, and SPECIFICATION.md §1.5 for the BOM version this pins.
- **Web stack mix (confirmed 2026-09-01):** production services split across both MVC-only and
  WebFlux-only, independent of Spring Boot generation — §2.4's "equivalent guarantees on both
  stacks" is not negotiable for either starter.

### 2.10 Versioning and distribution
- The library follows Semantic Versioning (`MAJOR.MINOR.PATCH`). Breaking changes to the public API (including the `LogSituationCode` contract, auto-configuration property names, or default output formats) require a major version bump.
- The library is published to an internal Nexus Repository (resolved 2026-09-01 — see §4). Consuming services depend on it as a normal versioned Maven dependency; exact base URL, instance topology, and credentials are an environment-specific implementation detail, not a design decision (see SPECIFICATION.md §13).

### 2.11 Project coordinates
- **GroupId / base package:** `com.llizzard.logcontrol` — the company's reverse-domain root (`com.llizzard`) is used per standard Maven convention, guaranteeing the library's coordinates and package namespace cannot collide with any other org's artifacts once published to a shared repository.
- **ArtifactIds:** three modules, per §2.12's multi-generation split — `log-control-core` (shared logic, no Spring/Jakarta/Jackson dependency), `log-control-spring-boot-starter` (Spring Boot 3.5.x line), and `log-control-spring-boot4-starter` (Spring Boot 4.1.x line, pinned to 4.1.1). Per Spring Boot's own documented guidance for third-party starters, both starter artifactIds deliberately avoid any `spring-boot-*` prefix (reserved for official Spring Boot artifacts) and instead lead with the project name.
- **Module split history:** an original single-module plan (`log-control-spring-boot-starter` alone, with a possible future `log-control-spring-boot` + `-starter` aggregator split if auto-configuration complexity ever warranted it) was superseded 2026-09-01 by §2.12's multi-generation split, which is needed for dependency-compatibility reasons regardless of complexity. Either starter can still separately apply a logic/aggregator split internally later if warranted.
- **Configuration property namespace:** properties should be prefixed `logcontrol.*` (never `spring.*`, `server.*`, or `management.*`), for the same collision-avoidance reasoning behind the groupId choice. The full property list remains an implementation detail (see SPECIFICATION.md §9).

### 2.12 Multi-generation Spring Boot support (resolved 2026-09-01; reaffirmed 2026-09-01)

- The library is consumed by a MIX of services today: some already running in production on
  Spring Boot 4.1.1, others still on Spring Boot 3.5.x with a migration to 4.x planned but not yet
  complete. Both generations must be released together at v1 — neither is deferred to a later
  release. (An earlier same-day pass briefly narrowed v1 to 3.5.x only, deferring Boot 4 support to
  a later phase, on the mistaken assumption the organization was 3.5.x-only today. That was wrong —
  real production apps already run on Spring Boot 4.1.1 — and has been reverted the same day.) The
  mix also spans both web stacks — MVC-only and WebFlux-only services exist in production
  (confirmed 2026-09-01) — independent of generation.
- These two generations pull in incompatible major versions of some of the same transitive
  dependencies the library touches directly — confirmed: Jackson 3 vs Jackson 2 (via
  `logstash-logback-encoder` 9.x, which dropped support for Jackson below 3.0.0 entirely, vs 8.x
  for the JSON encoder) and Jakarta EE 11/Servlet 6.1 vs EE 10/Servlet 6.0 for the MVC filter. A
  single starter jar cannot declare a `logstash-logback-encoder`/Jackson version that works for
  both a Jackson-2 app and a Jackson-3 app at once — that alone is a sufficient blocker for a
  single jar to satisfy both classpaths.
  **Correction (2026-09-01):** an earlier pass of this document also cited "Micrometer 2.x vs 1.x
  `context-propagation`" as a third incompatible dependency. That was checked and is not currently
  true: as of 2026-09-01, `io.micrometer:context-propagation` has not released a 2.x (latest is
  1.2.1) and `micrometer-core` is likewise still 1.x — both generations use the same Micrometer
  major version. This doesn't change the decision to split (Jackson alone is sufficient reason),
  but it does mean the WebFlux context-propagation code is not known to need per-generation
  divergence — see SPECIFICATION.md §6.
- Decision: split into a framework-agnostic `log-control-core` module (situation-code contract,
  fallback logic, validation, text-format layout — no Spring/Jakarta/Jackson dependency) plus two
  generation-specific starter modules that each depend on core: `log-control-spring-boot-starter`
  (existing artifactId, tracks the 3.5.x line) and `log-control-spring-boot4-starter` (new
  artifactId, tracks the 4.1.x line, pinned to 4.1.1). Consuming services declare whichever starter matches their
  own Spring Boot generation; core logic (situation codes, fallback behavior, text formatting) is
  written and tested once and shared by both. Both starters ship together as v1, in parallel —
  neither is a stub or a later addition.
- **Build order (confirmed 2026-09-01):** `log-control-core` is coded first regardless, since both
  starters depend on it. Of the two starters, `log-control-spring-boot-starter` (3.5.x) is coded
  first, followed by `log-control-spring-boot4-starter` (4.1.1) — this is a coding-sequence
  decision only, driven by 3.5.x being the line most in need of migrating off, and does not change
  the release commitment above: both starters still ship together in the same v1 release, not as
  separate versions.
- This supersedes §2.11's "future module split" as a *complexity* trigger — the split is now
  needed for generation-compatibility reasons regardless of how complex the auto-configuration
  logic gets. Either generation-specific starter can still later apply §2.11's own logic/aggregator
  split internally if it grows complex enough to warrant it.
- CI/testing implication: two build pipelines, each matrixed only within its own generation (3.5.x
  today, with room for a later 3.x-line bump if a new final minor ever ships; 4.1.x for the other
  line, pinned to 4.1.1 today with room to matrix later 4.x minors) — never across the 3.x/4.x
  boundary in one job. Both pipelines run for every change, since both generations are in active
  use. Coding still proceeds 3.5.x-starter-first per the build-order bullet above; CI for both
  pipelines is expected to exist before the v1 release regardless of coding order.
- SPECIFICATION.md is structured to reflect this split — package layout, class contracts, POM
  dependency tables, auto-configuration tables, config properties, and the test/CI matrix are all
  written per-module across `log-control-core` and the two starters.

## 3. Scope

**In scope for v1:** the logging API and context-enrichment mechanism described above, for both MVC and WebFlux, with configurable text/JSON output via Logback, distributed as a Spring Boot starter.

**Out of scope for v1 (non-goals):**
- Being a general-purpose logging framework replacement (it augments SLF4J/Logback usage, it does not replace them).
- Log shipping, storage, or aggregation infrastructure (e.g. standing up ELK/Datadog — the library only produces log lines in a format those systems can consume).
- Business-level validation or enforcement of what a "correct" situation code is beyond uniqueness within the enum contract.
- Support for logging backends other than Logback, or web stacks other than MVC/WebFlux.

## 4. Open items / assumptions to confirm

Original kickoff open items — all resolved during the 2026-09-01 full-spec pass (see
SPECIFICATION.md for the concrete design behind each):

- **Internal artifact repository target:** resolved 2026-09-01 — Nexus Repository. `pom.xml`'s
  `<distributionManagement>` targets Nexus-conventional release/snapshot repository IDs; the exact
  base URL, instance (dev/staging/prod), and credentials remain an environment-specific detail to
  fill in at implementation time, not a blocking design choice. See SPECIFICATION.md §13.
- **Testing/quality gates:** resolved — JaCoCo minimum 80% instruction / 70% branch coverage per
  module, plus a mandatory integration/unit test list covering both MVC and WebFlux context
  propagation, situation-code validation, fallback visibility, JSON/text output, and
  no-sensitive-data-leakage. See SPECIFICATION.md §12.
- **Situation code format:** resolved — enforced `SERVICE-CATEGORY-NNN` pattern (regex in
  SPECIFICATION.md §3), overridable per service, enforced by fail-fast startup validation.
- **Fallback-code visibility mechanism:** resolved — WARN log (dedicated internal logger) +
  Micrometer counter, both independently toggleable, combined via `CompositeFallbackUsageHandler`.
  See SPECIFICATION.md §7.

No open items remain blocking implementation as of 2026-09-01.
