# Log-Control Starter — Technical Specification (v1)

**Status:** Ready for implementation
**Depends on:** `CONSTITUTION.md` (non-negotiables, including §2.12) + kickoff Q&A resolutions
(2026-09-01)
**Scope:** v1 supports **both Spring Boot generations now, released together** — the organization
runs a mix of services: some already on Spring Boot 4.1.1 in production, others still on Spring
Boot 3.5.x with a migration to 4.x planned but not yet complete. Both starters described below are
v1 scope. Coding order is sequenced — `log-control-spring-boot-starter` (3.5.x) first, then
`log-control-spring-boot4-starter` (4.1.1), see §12.3 — but both still ship together in the same
v1 release; neither is a later, separate release.
**Audience:** the engineer or coding agent implementing v1 across the library's three Maven
modules.

This document turns the constitution's decisions into concrete package layout, class contracts,
algorithms, configuration keys, and acceptance criteria. Where the constitution states *what*
must be true, this document states *how* it is built. Anything left ambiguous here should be
resolved by re-reading `CONSTITUTION.md` §2 first; if the two conflict, the constitution wins and
the conflict should be flagged back to the project owner rather than silently resolved.

---

## 0. Baseline — resolved 2026-09-01, corrected 2026-09-01, reaffirmed 2026-09-01

The library targets **two separate Spring Boot generations at once**, both real today
(constitution §2.12) — not a "current generation now, future generation later" split. The
organization runs a mix of services: some already on Spring Boot 4.1.1 in production, others still
on Spring Boot 3.5.x with a migration to 4.x planned but not yet complete. Both need this library
now, released together; only the coding order between the two starters is sequenced (§12.3).

- **3.x line, floor Spring Boot 3.5.x** (the last released 3.x minor — past its own OSS
  end-of-life as of 2026-06-30, but still running production services; 3.2 through 3.4 are also
  no longer OSS-supported and are not targeted).
- **4.x line, floor Spring Boot 4.1.x** — pinned to Spring Boot 4.1.1, the exact minor
  confirmed running in production as of 2026-09-01. Jackson 3 required, `logstash-logback-encoder`
  9.x vs 8.x, Jakarta EE 11/Servlet 6.1 vs EE 10/Servlet 6.0, JUnit Jupiter 6 baseline.

Those two lines pull incompatible major versions of some of the same transitive dependencies the
library touches directly, so a single jar cannot serve both — see the correction below for the
confirmed list. The library is three Maven modules — a framework-agnostic `log-control-core`,
plus two thin generation-specific starters, `log-control-spring-boot-starter` (3.5.x line) and
`log-control-spring-boot4-starter` (4.1.x line, pinned to 4.1.1) — each described below,
released together under one version, both built and tested as v1 scope (coded in sequence,
3.5.x starter first — §12.3).

**Correction (2026-09-01):** an earlier pass of this document also listed "Micrometer 2.x
`context-propagation` vs 1.x" as a third incompatible dependency, which is what originally drove
§6's "real API risk, verify per generation" framing for the WebFlux integration. That was checked
and is not currently true: `io.micrometer:context-propagation` has not released a 2.x (latest is
1.2.1) and `micrometer-core` is likewise still 1.x — both starters depend on the same Micrometer
major version. The confirmed, sufficient reason for the module split is Jackson 3 vs 2 (via
`logstash-logback-encoder`, which dropped support for Jackson below 3.0.0 entirely in 9.x) plus
the Jakarta EE 11/Servlet 6.1 vs EE 10/Servlet 6.0 baseline bump. §6 below is written
accordingly — the WebFlux bridge is expected to be identical source between the two starters,
kept in each starter for classpath-isolation reasons rather than because it's known to differ.

**Scope note (2026-09-01):** an earlier same-day pass briefly narrowed v1 to the 3.5.x starter
only, deferring Spring Boot 4 support to a later phase, on the mistaken assumption the
organization was 3.5.x-only today. That was wrong — real production apps already run on Spring
Boot 4.1.1 — and has been reverted. Both starters are v1 scope, released together; the confirmed
coding order is 3.5.x starter first, then the 4.1.1 starter (§12.3), which does not change that
release commitment.

---

## 1. Module layout

### 1.1 Why three modules, and why not `spring-boot-starter-parent`

`log-control-core` has **zero dependency on Spring, Jakarta EE, Jackson, or Micrometer** — it is
plain Java 17 + `slf4j-api` only, and is compiled once and reused unchanged by both starters. Only
the classpath-sensitive integration glue (servlet/reactive filters, the Micrometer
context-propagation bridge, the `@ConfigurationProperties` binding shell, the JSON encoder
dependency) is generation-specific and lives in one starter or the other, never in core.

Because the project is a multi-module reactor, none of the three modules extends
`org.springframework.boot:spring-boot-starter-parent` directly — a Maven module can only have one
`<parent>`, and that slot is used for the reactor root pom instead (the reactor root itself has
**no Spring Boot parent at all** — plain `packaging=pom`, housing only shared
`<properties>`/`<build><pluginManagement>` defaults for the compiler/surefire/jacoco plugin
versions that `spring-boot-starter-parent` would otherwise have supplied). Each starter instead
**imports** `org.springframework.boot:spring-boot-dependencies` as a BOM in
`<dependencyManagement>`, at the version matching its generation, so its own and its transitive
dependency versions stay aligned with that generation's Spring Boot release train without pinning
versions by hand everywhere. This is also the conventional pattern for a *library* as opposed to
an *application*: `spring-boot-starter-parent` bundles application-oriented plugin defaults a
library doesn't want, and — critically here — pinning the reactor root's own `<parent>` to
`spring-boot-starter-parent` at any single version would leak that generation's plugin/dependency
defaults onto the *other* starter too, which is exactly what the module split exists to avoid.

```
log-control/                                    (reactor root)
├── pom.xml                                      packaging=pom, <modules> lists all three below
├── CONSTITUTION.md
├── SPECIFICATION.md
├── README.md
├── log-control-core/
├── log-control-spring-boot-starter/             Spring Boot 3.5.x line
└── log-control-spring-boot4-starter/            Spring Boot 4.1.x line
```

### 1.2 Versioning across the three modules

All three modules are released together under one shared version number (constitution §2.10's
SemVer applies to the release as a whole, not per module) — a given release ships
`log-control-core:X.Y.Z`, `log-control-spring-boot-starter:X.Y.Z`, and
`log-control-spring-boot4-starter:X.Y.Z` together. Consuming services never declare `core`
directly; it arrives transitively through whichever starter they depend on. A service picks
exactly one starter, matching its own Spring Boot generation, same adoption story either way.

### 1.3 `log-control-core` — coordinates and layout

```
groupId:     com.llizzard.logcontrol
artifactId:  log-control-core
packaging:   jar
parent:      (reactor root pom, not spring-boot-starter-parent)
```

```
log-control-core/
├── pom.xml
└── src/
    ├── main/java/com/llizzard/logcontrol/
    │   ├── core/
    │   │   ├── LogSituationCode.java
    │   │   ├── CoreSituationCode.java
    │   │   ├── ContextualLogger.java
    │   │   ├── ContextualLoggerImpl.java
    │   │   ├── ContextualLoggerFactory.java
    │   │   └── ContextualLoggerSettings.java
    │   ├── context/
    │   │   ├── RequestLogContext.java
    │   │   ├── RequestLogContextHolder.java
    │   │   ├── NoOpRequestLogContext.java
    │   │   └── LogControlMdcKeys.java
    │   ├── validation/
    │   │   ├── SituationCodeValidator.java
    │   │   └── SituationCodeValidationException.java
    │   └── fallback/
    │       ├── FallbackUsageHandler.java
    │       └── FallbackUsageWarnLogger.java
    └── test/java/com/llizzard/logcontrol/
        ├── core/ContextualLoggerImplTest.java
        ├── validation/SituationCodeValidatorTest.java
        └── fallback/FallbackUsageWarnLoggerTest.java
```

`pom.xml` dependencies: `org.slf4j:slf4j-api` (compile) and a test scope of `junit-jupiter` +
`assertj-core` only — no Spring, no Boot, no BOM import needed here at all. This module can be
built, tested, and versioned completely independently of either Spring Boot generation.

### 1.4 Starter module shape (applies to both `log-control-spring-boot-starter` and
`log-control-spring-boot4-starter`)

The two starters are **structurally identical** — same package names, same class names, same
file tree shape — because a consuming service only ever declares one of them, never both, so
there is no runtime classpath collision from the overlap. Template tree (shown once; both
starters follow it):

```
log-control-spring-boot{,4}-starter/
├── pom.xml
└── src/
    ├── main/
    │   ├── java/com/llizzard/logcontrol/
    │   │   ├── mvc/
    │   │   │   ├── ServletRequestLogContext.java
    │   │   │   └── LogControlMvcFilter.java
    │   │   ├── webflux/
    │   │   │   ├── ReactiveRequestLogContext.java
    │   │   │   ├── LogControlWebFilter.java
    │   │   │   └── RequestLogContextThreadLocalAccessor.java
    │   │   ├── validation/
    │   │   │   └── SituationCodeStartupValidator.java
    │   │   ├── fallback/
    │   │   │   ├── FallbackUsageMetrics.java
    │   │   │   └── CompositeFallbackUsageHandler.java
    │   │   └── autoconfigure/
    │   │       ├── LogControlProperties.java
    │   │       ├── LogControlAutoConfiguration.java
    │   │       ├── LogControlMvcAutoConfiguration.java
    │   │       ├── LogControlWebFluxAutoConfiguration.java
    │   │       ├── LogControlValidationAutoConfiguration.java
    │   │       └── LogControlMetricsAutoConfiguration.java
    │   └── resources/
    │       ├── META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
    │       └── META-INF/logcontrol/logback-include.xml
    └── test/
        ├── java/com/llizzard/logcontrol/
        │   ├── mvc/LogControlMvcIntegrationTest.java
        │   ├── webflux/LogControlWebFluxIntegrationTest.java
        │   ├── validation/SituationCodeStartupValidatorTest.java
        │   ├── fallback/FallbackUsageMetricsTest.java
        │   ├── logback/JsonOutputFormatTest.java
        │   ├── logback/TextOutputFormatTest.java
        │   ├── security/NoSensitiveDataLeakageTest.java
        │   └── testsupport/
        │       ├── ListAppenderTestSupport.java
        │       ├── mvc/{TestMvcApplication,TestController}.java
        │       └── webflux/{TestWebFluxApplication,TestController}.java
        └── resources/application-test.yml
```

### 1.5 What actually differs between the two starters

| | `log-control-spring-boot-starter` | `log-control-spring-boot4-starter` |
|---|---|---|
| artifactId | `log-control-spring-boot-starter` (existing, unchanged) | `log-control-spring-boot4-starter` (new) |
| Spring Boot BOM imported | `spring-boot-dependencies:3.5.x` | `spring-boot-dependencies:4.1.1` (pinned; bump within the 4.1.x line as later patches release) |
| Servlet / Jakarta EE | EE 10, Servlet 6.0 | EE 11, Servlet 6.1 |
| JSON encoder | `net.logstash.logback:logstash-logback-encoder` **8.x** (Jackson 2) | `net.logstash.logback:logstash-logback-encoder` **9.x** (Jackson 3; note the Jackson groupId itself also moved, `com.fasterxml.jackson.*` → `tools.jackson.*` — irrelevant to this library's own code since it never imports Jackson directly, but relevant if anyone adds an explicit Jackson dependency later) |
| Context propagation | `io.micrometer:context-propagation` **1.x** | `io.micrometer:context-propagation` **1.x** — same major version on both starters (corrected 2026-09-01, see §0); confirm the exact minor each generation's BOM manages at implementation time |
| Test framework | JUnit Jupiter 5 (via `spring-boot-starter-test` 3.5.x) | JUnit Jupiter 6 (mandatory in Boot 4's test stack; JUnit 4/vintage deprecated) |
| `logback-include.xml` content | identical XML to the 4.x column — the encoder class name (`net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder`) is unchanged between logstash-logback-encoder 8.x and 9.x; only the POM-managed Jackson version underneath it differs | see left column |
| Java source under `mvc/`, `webflux/`, `validation/`, `fallback/`, `autoconfigure/` | same class/package names as the 4.x column; expected to be byte-identical source in most cases now that Micrometer is confirmed unchanged across generations (§0) — the only currently-known per-generation difference is at the POM level (§1.7: `logstash-logback-encoder`/Jackson version, BOM version), not in this Java source. **Verify at implementation time** against each generation's actual Spring Framework 6.2 vs 7.0 API surface (`WebFilter`, `ServerWebExchange`, `HandlerMapping.BEST_MATCHING_PATTERN_ATTRIBUTE`, `FilterRegistrationBean`, `@ConditionalOnClass`/`@ConditionalOnWebApplication`) before assuming full parity | see left column |

### 1.6 POM dependencies — `log-control-core`

| Dependency | Scope | Purpose |
|---|---|---|
| `org.slf4j:slf4j-api` | compile | `ContextualLogger`'s underlying calls; MDC push/pop |
| `org.junit.jupiter:junit-jupiter`, `org.assertj:assertj-core` | test | pure unit tests, no Spring context needed |

### 1.7 POM dependencies — each starter (version column per §1.5)

| Dependency | Scope | Purpose |
|---|---|---|
| `com.llizzard.logcontrol:log-control-core` | compile | shared logic, same version as the starter (§1.2) |
| `spring-boot-starter` | compile | base autoconfig support, Logback transitively |
| `spring-boot-autoconfigure` | compile | `@AutoConfiguration`, `@ConditionalOn*` |
| `spring-boot-configuration-processor` | optional/compile | IDE metadata for `logcontrol.*` properties |
| `spring-boot-starter-web` | provided/optional | MVC types; gated by `@ConditionalOnClass` |
| `spring-boot-starter-webflux` | provided/optional | WebFlux types; gated by `@ConditionalOnClass` |
| `net.logstash.logback:logstash-logback-encoder` | compile | JSON structured encoder — **8.x on the 3.5.x starter, 9.x on the 4.x starter** (§1.5) |
| `org.codehaus.janino:janino` | compile | Logback `<if>` conditional config (format switch) |
| `io.micrometer:context-propagation` | compile | Reactor Context → ThreadLocal/MDC bridging — **same 1.x major on both starters** (§1.5, corrected 2026-09-01); confirm exact minor per generation's BOM at implementation time |
| `io.micrometer:micrometer-core` | provided/optional | fallback-usage counter; gated by `@ConditionalOnClass` |
| `spring-boot-starter-test`, `reactor-test`, `spring-boot-starter-web`, `spring-boot-starter-webflux` | test | integration test harnesses for both stacks, at that generation's version |

`logstash-logback-encoder` is the one dependency in this table confirmed to need a different
major version per generation — double-check it against the actual `spring-boot-dependencies` BOM
for each generation at implementation time, and confirm whether the BOM manages a compatible
version automatically or whether an explicit `<version>` override is needed. `context-propagation`
is currently the same major version (1.x) on both generations — worth a quick version-alignment
check, but not expected to need any code-level handling.

---

## 2. Core API — module: `log-control-core`

### 2.1 `LogSituationCode`

```java
package com.llizzard.logcontrol.core;

public interface LogSituationCode {
    String code();          // must match the configured pattern, see §3
    String description();   // human-readable, shown in text output and validation errors
}
```

Consuming services implement this with their own enum, e.g.:

```java
public enum OrderSituationCode implements LogSituationCode {
    ORDER_CREATED("ORDR-FLW-001", "Order successfully created"),
    ORDER_VALIDATION_FAILED("ORDR-VAL-002", "Order failed input validation"),
    PAYMENT_GATEWAY_TIMEOUT("ORDR-EXT-014", "Timed out calling the payment gateway");

    private final String code;
    private final String description;
    // constructor / getters implementing LogSituationCode
}
```

### 2.2 `CoreSituationCode` (built-in fallback)

```java
package com.llizzard.logcontrol.core;

public enum CoreSituationCode implements LogSituationCode {
    UNSPECIFIED("SYS-FALLBACK-000", "No situation code was resolved for this log call; the call site needs to be fixed to supply one.");
}
```

`SYS` is a reserved service mnemonic — consuming services must not register their own codes
under `SYS-*` (documented in the README; not mechanically enforced in v1).

### 2.3 `ContextualLogger`

The library's only supported logging entry point. There is deliberately no overload that omits
the code argument (constitution §2.3).

```java
package com.llizzard.logcontrol.core;

public interface ContextualLogger {
    void trace(LogSituationCode code, String message, Object... args);
    void debug(LogSituationCode code, String message, Object... args);
    void info (LogSituationCode code, String message, Object... args);
    void warn (LogSituationCode code, String message, Object... args);
    void error(LogSituationCode code, String message, Object... args);
    void error(LogSituationCode code, String message, Throwable throwable, Object... args);

    boolean isTraceEnabled();
    boolean isDebugEnabled();
    boolean isInfoEnabled();
    boolean isWarnEnabled();
    boolean isErrorEnabled();
}
```

Message/args follow SLF4J's `{}` placeholder convention and are passed straight through to the
underlying SLF4J call, preserving lazy argument evaluation when the level is disabled.

### 2.4 `ContextualLoggerFactory`

```java
package com.llizzard.logcontrol.core;

public final class ContextualLoggerFactory {
    public static ContextualLogger getLogger(Class<?> callerClass) { ... }
    public static ContextualLogger getLogger(String name) { ... }

    // called exactly once, at startup, by each starter's LogControlAutoConfiguration — see §2.6
    public static void configure(ContextualLoggerSettings settings, FallbackUsageHandler fallbackUsageHandler) { ... }
}
```

Mirrors `org.slf4j.LoggerFactory` on purpose so adoption is a mechanical find-replace:

```java
private static final ContextualLogger log = ContextualLoggerFactory.getLogger(OrderService.class);
...
log.info(OrderSituationCode.ORDER_CREATED, "Order {} created for customer {}", orderId, customerId);
```

**Test-isolation note:** `configure(...)` sets static state shared by the whole JVM. Each
starter's integration test suite (§12.2) boots several separate `@SpringBootTest` contexts in the
same module — harmless as long as they all configure the *same effective settings*. Any test that
deliberately overrides a non-default `logcontrol.*` property (e.g.
`logcontrol.context.include-raw-path=false`) must run in its own Maven Surefire fork
(`reuseForks=false` for that test, or an equivalent isolation mechanism) so it can't race with, or
leave state for, a test class expecting the defaults — in either starter module independently.

### 2.5 `ContextualLoggerImpl` — per-call algorithm

This is the crux of the "every log line carries all three fields" guarantee, and it is the same
code path for both MVC and WebFlux (§5–§6 only differ in how `RequestLogContext` gets populated),
and the same code path for both Spring Boot generations — nothing in this class touches Spring,
Jakarta, Jackson, or Micrometer.

```java
private void log(Level level, LogSituationCode code, Throwable t, String message, Object[] args) {
    LogSituationCode resolved = code;
    boolean isFallback = (code == null);
    if (isFallback) {
        resolved = CoreSituationCode.UNSPECIFIED;
    }

    RequestLogContext ctx = RequestLogContextHolder.current(); // never null (§4)

    MDC.put(LogControlMdcKeys.HTTP_METHOD, ctx.httpMethod());
    MDC.put(LogControlMdcKeys.ENDPOINT, ctx.resolveEndpoint());
    if (settings.includeRawPath()) {
        MDC.put(LogControlMdcKeys.REQUEST_PATH, ctx.rawPath());
    }
    MDC.put(LogControlMdcKeys.SITUATION_CODE, resolved.code());
    MDC.put(LogControlMdcKeys.SITUATION_DESCRIPTION, resolved.description());
    try {
        if (isFallback) {
            // callerClassName is simply this ContextualLoggerImpl's own bound logger name (the
            // Class<?> passed to ContextualLoggerFactory.getLogger(...) at creation time) — no
            // separate stack walk needed; §7's FallbackUsageWarnLogger and FallbackUsageMetrics
            // both receive this same precomputed string, so "which class triggered the fallback"
            // is answered one way, consistently, not by two different mechanisms.
            fallbackUsageHandler.onFallbackUsed(ctx, callerClassName);
        }
        // SLF4J 2.x fluent API; delegates to the wrapped org.slf4j.Logger
        slf4jLogger.atLevel(level).setCause(t).log(message, args);
    } finally {
        MDC.remove(LogControlMdcKeys.HTTP_METHOD);
        MDC.remove(LogControlMdcKeys.ENDPOINT);
        MDC.remove(LogControlMdcKeys.REQUEST_PATH);
        MDC.remove(LogControlMdcKeys.SITUATION_CODE);
        MDC.remove(LogControlMdcKeys.SITUATION_DESCRIPTION);
    }
}
```

Context is fetched and pushed into MDC **at the point of each log call**, not once per request.
This is required for WebFlux correctness (a request's logical flow hops across event-loop
threads; MDC is thread-local) and is deliberately used uniformly for MVC too, so both stacks run
through one implementation.

A `null` code is the only way a call can "reach the logging layer without a resolvable code"
(constitution §2.3) since the method signature otherwise requires one — this is where the guarded
fallback substitution and the fallback-visibility machinery (§7) both hook in.

### 2.6 `ContextualLoggerSettings` — the core/starter seam

`ContextualLoggerImpl` needs exactly one piece of external configuration
(`includeRawPath()`), plus the `FallbackUsageHandler` it calls on fallback. Neither can be a
Spring `@ConfigurationProperties` object, because that annotation and its binding machinery live
in `spring-boot`, and core has no dependency on Spring at all (§1.1). Instead:

```java
package com.llizzard.logcontrol.core;

public interface ContextualLoggerSettings {
    boolean includeRawPath();
}
```

Each starter's `LogControlProperties` (§9, §10 — a normal `@ConfigurationProperties` class,
duplicated per starter like everything else under `autoconfigure/`) implements this interface (or
adapts to it with a tiny wrapper) and is handed to `ContextualLoggerFactory.configure(...)` once,
by that starter's `LogControlAutoConfiguration`, at application startup. This is the one seam
where a starter reaches into core with a plain-Java object instead of core reaching out to Spring.

---

## 3. Situation code format and validation

**Pattern (enforced by default):** `SERVICE-CATEGORY-NNN`, regex
`^[A-Z0-9]{2,10}-[A-Z0-9]{2,10}-\d{3}$`

- `SERVICE` — short mnemonic for the owning service (e.g. `ORDR`, `PMT`, `AUTH`). `SYS` is
  reserved for the library's own fallback code.
- `CATEGORY` — logical grouping within the service (suggested starter set, non-enforced:
  `FLW` general flow, `VAL` validation, `DB` persistence, `EXT` external call, `SEC` security).
- `NNN` — three-digit zero-padded sequence, unique within `(SERVICE, CATEGORY)`.

Example: `ORDR-VAL-003`.

### 3.1 `SituationCodeValidator` — module: `log-control-core`

Pure function, no Spring dependency, easy to unit test:

```java
package com.llizzard.logcontrol.validation;

public final class SituationCodeValidator {
    public ValidationResult validate(Collection<? extends LogSituationCode> codes, Pattern pattern) {
        // 1. every code.code() matches `pattern`
        // 2. no two entries share the same code() value
        // returns a ValidationResult with a list of human-readable violation messages (empty = valid)
    }
}
```

### 3.2 `SituationCodeStartupValidator` — module: each starter (duplicated)

This class is Spring lifecycle wiring around §3.1's pure algorithm, so it lives in each starter,
not in core — but it is expected to be identical, or nearly so, between the two starters, since
the Spring Framework/Boot APIs it uses (`ApplicationListener`, `ApplicationReadyEvent`,
`ClassPathScanningCandidateComponentProvider`, `AutoConfigurationPackages`) are long-stable, low-
level Spring Boot APIs. Confirm this assumption at implementation time; if Spring Framework 7 did
change any of these signatures, fix the 4.x-starter copy independently.

`SituationCodeStartupValidator` runs as an `ApplicationListener<ApplicationReadyEvent>`
(alternatively a `SmartInitializingSingleton` if fail-fast needs to happen before readiness):

1. Discover candidate enum classes implementing `LogSituationCode` via
   `ClassPathScanningCandidateComponentProvider` with an `AssignableTypeFilter(LogSituationCode.class)`
   include filter, scoped to `logcontrol.situation-code.base-packages` if set, else the packages
   returned by `org.springframework.boot.autoconfigure.AutoConfigurationPackages.get(beanFactory)`
   (the consuming app's own auto-detected base package — this is the same mechanism Spring Data /
   JPA starters use to find `@Entity` classes without extra configuration).
2. For each discovered class, get its constants via `clazz.getEnumConstants()` (cast to
   `LogSituationCode[]`) — **not** `EnumSet.allOf(...)`, which needs a compile-time-known enum
   type parameter and doesn't fit a `Class<?>` obtained by reflection. Pass the resulting array to
   `SituationCodeValidator.validate(Arrays.asList(constants), pattern)`.
3. If any violation is found, throw `SituationCodeValidationException` (from core, §3.1's package)
   listing every offending constant and reason (bad pattern vs. duplicate code) — this fails
   application startup.

Controlled by:
- `logcontrol.situation-code.validation-enabled` (default `true`)
- `logcontrol.situation-code.pattern` (default the regex above — override if a service needs a
  different convention while keeping the rest of the library's guarantees)
- `logcontrol.situation-code.base-packages` (default: auto-detected)

---

## 4. Request context capture — shared abstraction — module: `log-control-core`

```java
package com.llizzard.logcontrol.context;

public interface RequestLogContext {
    String httpMethod();     // e.g. "GET"; fixed at request entry
    String rawPath();        // resolved raw path, fixed at request entry
    String resolveEndpoint(); // best-matching route template if known *right now*, else rawPath
}
```

`resolveEndpoint()` is intentionally **lazy** — it re-reads the underlying request/exchange
attribute on every call rather than caching a value at construction time, because the route
template is not yet known when the request first enters the filter and only becomes available
once Spring's handler mapping has run (still before any application/controller code executes).

`RequestLogContextHolder` is a `ThreadLocal<RequestLogContext>` with `set`/`clear`/`current()`;
`current()` never returns `null` — absent context (e.g. logging from a `@Scheduled` job or during
startup, outside any request) returns `NoOpRequestLogContext`, whose three accessors all return
`"N/A"`. This keeps constitution §2.1's "no plain logging mode that skips them" true even outside
a request: the fields are always present on the line, sometimes with an explicit sentinel value
instead of a real one.

```java
package com.llizzard.logcontrol.context;

public final class LogControlMdcKeys {
    public static final String HTTP_METHOD = "httpMethod";
    public static final String ENDPOINT = "endpoint";
    public static final String REQUEST_PATH = "requestPath";       // secondary, §2.1
    public static final String SITUATION_CODE = "situationCode";
    public static final String SITUATION_DESCRIPTION = "situationDescription";
}
```

Both `mvc/ServletRequestLogContext` and `webflux/ReactiveRequestLogContext` (§5, §6, in each
starter) implement this same core interface, so `ContextualLoggerImpl` never needs to know which
stack — or which Spring Boot generation — it's running under.

---

## 5. Request context capture — MVC — module: each starter (duplicated)

One component only — no `HandlerInterceptor` is needed, because `resolveEndpoint()` reads the
route-template request attribute live at log time, and that attribute is already populated by
the time any controller/service code runs.

```java
package com.llizzard.logcontrol.mvc;

public class LogControlMvcFilter extends OncePerRequestFilter implements Ordered {
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain) {
        RequestLogContextHolder.set(new ServletRequestLogContext(request));
        try {
            chain.doFilter(request, response);
        } finally {
            RequestLogContextHolder.clear();
        }
    }
}
```

```java
package com.llizzard.logcontrol.mvc;

class ServletRequestLogContext implements RequestLogContext {
    private final HttpServletRequest request;
    private final String method;    // captured once, at construction
    private final String rawPath;   // captured once, at construction

    public String httpMethod() { return method; }
    public String rawPath() { return rawPath; }
    public String resolveEndpoint() {
        Object pattern = request.getAttribute(HandlerMapping.BEST_MATCHING_PATTERN_ATTRIBUTE);
        return pattern != null ? pattern.toString() : rawPath; // 404 / no route matched yet
    }
}
```

`jakarta.servlet.http.HttpServletRequest`/`HttpServletResponse` come from Jakarta EE 10 (Servlet
6.0) on the 3.5.x starter and Jakarta EE 11 (Servlet 6.1) on the 4.x starter — same package names
(`jakarta.servlet.*`) on both, so this source is expected to compile unchanged against either;
confirm at implementation time rather than assuming, since it is compiled and tested separately
per starter regardless.

Registered via `FilterRegistrationBean<LogControlMvcFilter>` at order
`logcontrol.mvc.filter-order` (default `Ordered.HIGHEST_PRECEDENCE + 10`, leaving headroom for
anything that legitimately needs to run before it, e.g. a correlation-ID filter if one exists
outside this library's scope).

---

## 6. Request context capture — WebFlux — module: each starter (duplicated for classpath
isolation; source expected identical)

This is the highest-risk part of the implementation generally — get the thread-hop test in §12
passing before considering this section done, on both starters. `context-propagation` is
confirmed 1.x on both generations (§0), so this code is not known to need per-generation
divergence; it still lives in each starter's own module for classpath-isolation reasons (§1.1),
not because the Micrometer API is known to diverge. The smaller residual risk worth a quick check
at implementation time is whether Spring Framework 7 (Boot 4)'s versions of `WebFilter`,
`ServerWebExchange`, and `HandlerMapping.BEST_MATCHING_PATTERN_ATTRIBUTE` are unchanged from
Framework 6.2 (Boot 3.5) — not a Micrometer concern.

### 6.1 Mechanism

1. `LogControlWebFilter` writes a `ReactiveRequestLogContext` into the Reactor `Context` at
   subscription time, wrapping the request's `ServerWebExchange`.
2. `RequestLogContextThreadLocalAccessor` is registered with Micrometer's global
   `ContextRegistry` at auto-configuration time, bridging that Reactor Context entry into the
   same `RequestLogContextHolder` ThreadLocal that MVC uses.
3. `Hooks.enableAutomaticContextPropagation()` is invoked once at startup so that Reactor
   automatically re-establishes the correct ThreadLocal around every operator boundary/thread
   hop — without this, code running inside a `.flatMap()` after a thread switch (e.g. following
   an async downstream call) would not see the right context.

```java
package com.llizzard.logcontrol.webflux;

public class LogControlWebFilter implements WebFilter, Ordered {
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        RequestLogContext ctx = new ReactiveRequestLogContext(exchange);
        return chain.filter(exchange)
                .contextWrite(Context.of(RequestLogContext.class, ctx));
    }
}
```

```java
package com.llizzard.logcontrol.webflux;

class ReactiveRequestLogContext implements RequestLogContext {
    private final ServerWebExchange exchange;
    private final String method;   // captured once
    private final String rawPath;  // captured once

    public String resolveEndpoint() {
        Object pattern = exchange.getAttribute(HandlerMapping.BEST_MATCHING_PATTERN_ATTRIBUTE);
        return pattern != null ? pattern.toString() : rawPath;
    }
}
```

```java
package com.llizzard.logcontrol.webflux;

public class RequestLogContextThreadLocalAccessor implements ThreadLocalAccessor<RequestLogContext> {
    public Object key() { return RequestLogContext.class; }
    public RequestLogContext getValue() { return RequestLogContextHolder.current(); }
    public void setValue(RequestLogContext value) { RequestLogContextHolder.set(value); }
    public void reset() { RequestLogContextHolder.clear(); }
}
```

No Micrometer version reconciliation is needed here (§0) — `context-propagation` is 1.x on both
starters, so `ThreadLocalAccessor`, `ContextRegistry`, and
`Hooks.enableAutomaticContextPropagation()` should have identical signatures on both. This class
is expected to be a straight copy between the two starters.

Registered in `LogControlWebFluxAutoConfiguration`:

```java
@PostConstruct
void registerContextPropagation() {
    ContextRegistry.getInstance().registerThreadLocalAccessor(new RequestLogContextThreadLocalAccessor());
    Hooks.enableAutomaticContextPropagation(); // idempotent
}
```

### 6.2 Why exchange attributes, not raw strings, go into the Context

Reactor `Context` values are fixed at the point they're written. The route template becomes
known only *after* the filter has already returned its `Mono` chain (handler mapping happens
downstream, inside `chain.filter(exchange)`), so writing a resolved string into Context up front
is not possible. Instead the Context carries a stable **reference** (the exchange, via
`ReactiveRequestLogContext`), and `resolveEndpoint()` re-reads the exchange's mutable attribute
map — which *is* populated by the time application code executes — at each log call, exactly
mirroring the MVC design in §5.

---

## 7. Fallback visibility — split across core and each starter

`FallbackUsageHandler` is invoked exactly when `ContextualLoggerImpl` receives a `null` code
(§2.5), with the resolved `RequestLogContext` and the calling class name (the logger's own bound
name, per §2.5's note — not a separate stack walk). Two independent implementations exist, and
each starter's `CompositeFallbackUsageHandler` combines whichever are enabled into the single
`FallbackUsageHandler` that `ContextualLoggerFactory.configure(...)` expects:

- **`FallbackUsageHandler`** (interface) and **`FallbackUsageWarnLogger`** (its default
  implementation) — module: `log-control-core`. `FallbackUsageWarnLogger` logs a WARN through a
  dedicated internal SLF4J logger (`com.llizzard.logcontrol.fallback`), never through
  `ContextualLogger` itself, to avoid recursion. Message includes the calling class, HTTP method,
  and endpoint. Pure SLF4J, no Spring dependency, so it belongs in core alongside the interface;
  each starter's `LogControlAutoConfiguration` just wires it up
  (`logcontrol.fallback.warn-enabled`, default `true`).
- **`FallbackUsageMetrics`** — module: each starter (duplicated), package `fallback`, alongside
  core's `FallbackUsageHandler`/`FallbackUsageWarnLogger` (a package legitimately spanning core
  and starter jars, with no overlapping class names, the same way many Spring modules split a
  package across artifacts). Increments a Micrometer `Counter` named `logcontrol.fallback.usage`,
  tagged `httpMethod`, `endpoint`, and `originClass` (the calling class simple name — the same
  string `FallbackUsageWarnLogger` uses). Cardinality stays bounded because all three tags are
  drawn from a finite set of code paths and routes, not user input.
  `@ConditionalOnClass(MeterRegistry.class)`; controlled by `logcontrol.fallback.metrics-enabled`
  (default `true`). Duplicated, like the rest of the starter-layer classes, for
  classpath-isolation reasons — it's wired via `@ConditionalOnClass(MeterRegistry.class)` in each
  starter's own `autoconfigure` package, which is itself generation-specific — not because of a
  known Micrometer version difference (§0); the underlying `Counter`/`MeterRegistry` logic is
  expected to be identical source on both.
- **`CompositeFallbackUsageHandler`** — module: each starter (duplicated), package `fallback`.
  Combines whichever of the two handlers above are enabled into the one `FallbackUsageHandler`
  instance `LogControlAutoConfiguration` passes to `ContextualLoggerFactory.configure(...)`:

  ```java
  package com.llizzard.logcontrol.fallback;

  public final class CompositeFallbackUsageHandler implements FallbackUsageHandler {
      private final List<FallbackUsageHandler> delegates; // built by LogControlAutoConfiguration
                                                            // from whichever of FallbackUsageWarnLogger
                                                            // / FallbackUsageMetrics are enabled
      public void onFallbackUsed(RequestLogContext ctx, String callerClassName) {
          for (FallbackUsageHandler d : delegates) d.onFallbackUsed(ctx, callerClassName);
      }
  }
  ```

  An empty delegate list (both toggled off) is valid — fallback usage then simply produces no
  side effect beyond the substituted code itself.

---

## 8. Logback output — JSON and text — module: each starter (near-identical resource)

**Constraint to design around:** Logback initializes before the Spring `ApplicationContext`
exists, so auto-configuration beans cannot programmatically choose an encoder at runtime. The
constitution's "configuration change, never a code change" requirement (§2.5) is met using
Logback's own property-driven conditional config instead.

### 8.1 What each starter ships

`META-INF/logcontrol/logback-include.xml`, packaged inside the jar, containing both appenders
gated by a Logback property read from the Spring `Environment`. **This XML is the same on both
starters** — the encoder class name (`net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder`)
is unchanged between `logstash-logback-encoder` 8.x and 9.x; only the Jackson major version pulled
in underneath it differs, via each starter's own POM (§1.5, §1.7):

```xml
<included>
    <springProperty name="logcontrol.log.format" source="logcontrol.log.format" defaultValue="text"/>

    <if condition='property("logcontrol.log.format").equals("json")'>
        <then>
            <appender name="LOGCONTROL" class="ch.qos.logback.core.ConsoleAppender">
                <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
                    <providers>
                        <timestamp/>
                        <logLevel/>
                        <loggerName/>
                        <threadName/>
                        <message/>
                        <mdc/>            <!-- promotes httpMethod/endpoint/situationCode/etc to top-level JSON keys -->
                        <stackTrace/>
                    </providers>
                </encoder>
            </appender>
        </then>
        <else>
            <appender name="LOGCONTROL" class="ch.qos.logback.core.ConsoleAppender">
                <encoder>
                    <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [%thread] %logger{36} - method=%X{httpMethod} endpoint=%X{endpoint} code=%X{situationCode} - %msg%n%throwable</pattern>
                </encoder>
            </appender>
        </else>
    </if>

    <root level="INFO">
        <appender-ref ref="LOGCONTROL"/>
    </root>
</included>
```

### 8.2 What the consuming service must do

Add one line to its own `logback-spring.xml`:

```xml
<include resource="META-INF/logcontrol/logback-include.xml"/>
```

This is unchanged regardless of which starter a service depends on — same include path, same
property name. This must be documented prominently in the README as a required manual step —
Spring Boot does not auto-include a dependency's Logback config, and no auto-configuration bean
can work around that (the config file is parsed before any bean is available). Switching output
format is then purely `logcontrol.log.format=json|text` in `application.yml`, no code or XML
change.

Janino (`org.codehaus.janino:janino`) is a required (not optional) dependency of each starter
specifically to make the `<if>` conditional above work.

---

## 9. Configuration properties (`logcontrol.*`) — module: each starter (duplicated)

| Property | Type | Default | Purpose |
|---|---|---|---|
| `logcontrol.enabled` | boolean | `true` | master switch for all auto-configuration |
| `logcontrol.log.format` | `text`\|`json` | `text` | selects the Logback encoder, see §8 |
| `logcontrol.context.include-raw-path` | boolean | `true` | include the secondary raw-path MDC field |
| `logcontrol.situation-code.validation-enabled` | boolean | `true` | fail-fast startup validation, see §3.2 |
| `logcontrol.situation-code.pattern` | regex string | `^[A-Z0-9]{2,10}-[A-Z0-9]{2,10}-\d{3}$` | overridable per service |
| `logcontrol.situation-code.base-packages` | comma list | auto-detected | where to scan for `LogSituationCode` enums |
| `logcontrol.fallback.warn-enabled` | boolean | `true` | WARN log on fallback usage, see §7 |
| `logcontrol.fallback.metrics-enabled` | boolean | `true` | Micrometer counter on fallback usage, see §7 |
| `logcontrol.mvc.filter-order` | int | `Ordered.HIGHEST_PRECEDENCE + 10` | MVC filter registration order |
| `logcontrol.webflux.filter-order` | int | `Ordered.HIGHEST_PRECEDENCE + 10` | WebFilter registration order |

This property surface is identical on both starters by design — a service migrating from the
3.5.x starter to the 4.x starter (or vice versa) changes only its Maven dependency, never its
`application.yml`. `@ConfigurationProperties(prefix = "logcontrol")` on `LogControlProperties`,
with nested static classes `Log`, `Context`, `SituationCode`, `Fallback`, `Mvc`, `Webflux`
grouping the above; `LogControlProperties` implements (or adapts to) core's
`ContextualLoggerSettings` (§2.6).

---

## 10. Auto-configuration classes — module: each starter (duplicated)

Registered via `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
(Spring Boot 3.x+ mechanism — not the legacy `spring.factories`; unchanged in Boot 4). Each
starter ships its own `AutoConfiguration.imports` file listing its own package's classes.

| Class | Conditions | Registers |
|---|---|---|
| `LogControlAutoConfiguration` | `@ConditionalOnProperty(prefix="logcontrol", name="enabled", matchIfMissing=true)` | `LogControlProperties`; builds `CompositeFallbackUsageHandler` (§7) from whichever of `FallbackUsageWarnLogger`/`FallbackUsageMetrics` are enabled; calls `ContextualLoggerFactory.configure(...)` (§2.6) |
| `LogControlMvcAutoConfiguration` | `@ConditionalOnClass(DispatcherServlet.class)`, `@ConditionalOnWebApplication(type=SERVLET)` | `LogControlMvcFilter` bean via `FilterRegistrationBean` |
| `LogControlWebFluxAutoConfiguration` | `@ConditionalOnClass` on WebFlux types, `@ConditionalOnWebApplication(type=REACTIVE)` | `LogControlWebFilter` bean, registers `ThreadLocalAccessor`, calls `Hooks.enableAutomaticContextPropagation()` |
| `LogControlValidationAutoConfiguration` | `@ConditionalOnProperty(prefix="logcontrol.situation-code", name="validation-enabled", matchIfMissing=true)` | `SituationCodeStartupValidator` listener |
| `LogControlMetricsAutoConfiguration` | `@ConditionalOnClass(MeterRegistry.class)` | `FallbackUsageMetrics` |

No manual bean wiring should be required in a consuming service beyond the Maven dependency (one
starter, matching its Spring Boot generation), its own `LogSituationCode` enum, and the one
`logback-spring.xml` include line (§8.2) — matching constitution §2.8, unchanged by the module
split.

---

## 11. Security constraint (constitution §2.7) — applies to each starter

Neither `LogControlMvcFilter` nor `LogControlWebFilter` (in either starter) may read or log:
request/response bodies, the `Authorization` header, `Cookie`/`Set-Cookie` headers, or any other
header not explicitly needed to populate `httpMethod`/`rawPath`/`endpoint`.
`ServletRequestLogContext` and `ReactiveRequestLogContext` must only ever touch `getMethod()`,
path/route attributes, and nothing else off the request/exchange object. This must have a
dedicated regression test (`NoSensitiveDataLeakageTest`, §12) **in each starter**, rather than
being left as an implicit property of "the code just doesn't do that" or assumed to carry over
from one generation to the other untested.

---

## 12. Testing and quality gates

### 12.1 `log-control-core` — pure unit tests, no Spring context

| Test | Proves |
|---|---|
| `SituationCodeValidatorTest` | Valid/invalid pattern cases, duplicate-code detection (the pure-logic half of the original single test — the Spring-startup half moved to §12.2). |
| `ContextualLoggerImplTest` | Fallback substitution (`null` → `CoreSituationCode.UNSPECIFIED`), MDC push-then-remove lifecycle per call (using a fake `RequestLogContext` and a fake `ContextualLoggerSettings`/`FallbackUsageHandler`), that no overload omits the code argument, and that `callerClassName` passed to the fallback handler is the logger's own bound name. |
| `FallbackUsageWarnLoggerTest` | WARN message content and that it goes through the dedicated internal logger, not `ContextualLogger` (no recursion). |

JaCoCo minimum **80% instruction / 70% branch coverage** for `com.llizzard.logcontrol..*` within
this module, enforced via `jacoco-maven-plugin`'s `check` goal bound to `verify`.

### 12.2 Each starter — integration tests, doubled per generation

Each of `log-control-spring-boot-starter` and `log-control-spring-boot4-starter` runs its own
full copy of these, spinning up a real embedded application against that starter's own Spring
Boot BOM version, and capturing Logback output via an in-memory `ListAppender` attached to the
root logger for assertions. Any test overriding a non-default `logcontrol.*` property must run in
its own Surefire fork (§2.4) to avoid static-state bleed from `ContextualLoggerFactory.configure`
across `@SpringBootTest` contexts in the same module:

| Test | Proves |
|---|---|
| `LogControlMvcIntegrationTest` | `@SpringBootTest(webEnvironment=RANDOM_PORT)` + `spring-boot-starter-web`; real HTTP calls via `TestRestTemplate` against a controller with a path-variable route; asserts `httpMethod`/`endpoint` (template, not raw path)/`situationCode` all present on the emitted log event. Includes a request to an unmapped path asserting graceful fallback to raw path (404 case). |
| `LogControlWebFluxIntegrationTest` | Same shape with `spring-boot-starter-webflux` + `WebTestClient`; **must** include a case where the log call happens inside a `.flatMap()` following a `Mono.delay()` (forces a thread hop) and still asserts correct context — this is the test that actually proves §6 works under *that generation's* Spring Framework/WebFlux runtime, not just that it compiles. |
| `SituationCodeStartupValidatorTest` | Via `ApplicationContextRunner`, that a bad-pattern or duplicate-code enum fails application startup with a clear message (the Spring-lifecycle half of the original single test, §3.2). |
| `FallbackUsageMetricsTest` | Micrometer counter incremented when a `null` code reaches `ContextualLoggerImpl`, with the tags specified in §7, against that generation's Micrometer version; also that `CompositeFallbackUsageHandler` correctly fans out to both delegates when both are enabled, and to neither when both are disabled. |
| `JsonOutputFormatTest` / `TextOutputFormatTest` | Each Logback encoder (unit-level, no full Spring context needed) produces well-formed JSON / the documented text pattern, and that all five MDC fields appear — the JSON test in particular exercises that starter's Jackson major version end to end. |
| `NoSensitiveDataLeakageTest` | Sends a request with `Authorization` and `Cookie` headers and a JSON body; asserts none of that content appears anywhere in MDC or the emitted log line. |

JaCoCo minimum **80% instruction / 70% branch coverage** applies within each starter module
independently.

### 12.3 CI shape

Two build pipelines, matrixed only *within* their own generation (constitution §2.12); both exist
and gate every change once implemented, since both starters are v1 scope — never across the
3.x/4.x boundary in one job:

- **3.5.x pipeline:** `mvn -pl log-control-core,log-control-spring-boot-starter -am verify`
  (room to extend the matrix if a later 3.x-line minor were ever to ship, though none is expected
  given the whole 3.x generation is past its final minor as of this writing).
- **4.x pipeline:** `mvn -pl log-control-core,log-control-spring-boot4-starter -am verify`,
  pinned to `4.1.1` today (matrixed across later 4.x minors as they ship).

Both pipelines build `log-control-core` fresh (`-am`, "also make"), so a change to core is
validated against both generations before merge, even though core itself has no Spring Boot
version dependency to matrix against.

**Build order (confirmed 2026-09-01):** `log-control-core` first, then `log-control-spring-boot-starter`
(3.5.x) is implemented and brought fully green before starting `log-control-spring-boot4-starter`
(4.1.1) — a coding-sequence choice driven by 3.5.x being the line most in need of migrating off.
This is an implementation-order decision only: both starters are still released together as part
of the same v1 version, and the 4.x pipeline above must exist and pass before that release ships,
not sometime after it.

---

## 13. Distribution

- SemVer per constitution §2.10, applied to the release as a whole (§1.2): any change to
  `LogSituationCode`, auto-config property names, or default output formats is a major bump,
  applied to all three modules together.
- The reactor root `pom.xml` includes a `<distributionManagement>` block targeting the internal
  Nexus Repository (resolved 2026-09-01 — constitution §2.10/§4), inherited by all three modules.
  Repository IDs follow Nexus's conventional release/snapshot split; only the base URL is still a
  placeholder, to be filled in with the organization's actual Nexus instance URL:

```xml
<distributionManagement>
    <repository>
        <id>nexus-releases</id>
        <url>REPLACE_WITH_NEXUS_RELEASES_URL</url>
    </repository>
    <snapshotRepository>
        <id>nexus-snapshots</id>
        <url>REPLACE_WITH_NEXUS_SNAPSHOTS_URL</url>
    </snapshotRepository>
</distributionManagement>
```

  Credentials are supplied via `settings.xml` `<server>` entries matching the same IDs (env-var
  or CI-secret backed), never committed to the repo. Nexus itself is decided; only the specific
  instance URL/credentials (which environment, which host) is left to fill in at implementation
  time — that is independent of everything else in this spec.
- Suggested CI stages (platform-agnostic, to be mapped onto whatever CI system is chosen):
  build → unit + integration tests (§12) → JaCoCo check → (on tagged release) deploy all three
  module artifacts to the repo above, both pipelines from §12.3 gating the same release tag.

---

## 14. Documentation deliverables for v1

- `README.md`: quickstart, with an explicit "which starter do I use" step up front — pick
  `log-control-spring-boot-starter` if the consuming service is on Spring Boot 3.5.x, or
  `log-control-spring-boot4-starter` if it's on Spring Boot 4.1.x — before the rest of the
  quickstart (dependency snippet, minimal `LogSituationCode` enum example, the required
  `logback-spring.xml` include line, which are identical either way), full property table (§9),
  and an explanation of why that one manual Logback include step exists.
- Javadoc on every public class/interface in `log-control-core`'s `core`, `context`, and
  `validation` packages, and on each starter's `autoconfigure` package.

---

## 15. Acceptance criteria checklist

| # | Criterion | Source |
|---|---|---|
| 1 | Every `ContextualLogger` call, on both MVC and WebFlux, on both starters, emits a line with non-empty `httpMethod`, `endpoint`, `situationCode` fields | Constitution §2.1 |
| 2 | No overload of `ContextualLogger` exists that omits the code argument | Constitution §2.3 |
| 3 | A `null`/unresolvable code substitutes `CoreSituationCode.UNSPECIFIED` and never throws or drops fields | Constitution §2.3 |
| 4 | Fallback usage produces a WARN log and increments a Micrometer counter, both independently toggleable, on both starters, via `CompositeFallbackUsageHandler` fan-out | Kickoff resolution (§7) |
| 5 | A situation-code enum with a bad-pattern or duplicate code fails application startup with a clear message, on both starters | Kickoff resolution (§3) |
| 6 | WebFlux context survives a thread hop, proven separately on each starter under that generation's Spring Framework/WebFlux runtime | Constitution §2.4 |
| 7 | Output format switches between JSON and text via `logcontrol.log.format` alone, no code change, on both starters | Constitution §2.5 |
| 8 | No request body, `Authorization`, or cookie data ever appears in MDC or log output, on both starters | Constitution §2.7 |
| 9 | Adopting the library requires only: the Maven dependency (one starter, matching the service's Spring Boot generation), a situation-code enum, and the Logback include line | Constitution §2.8 |
| 10 | JaCoCo gate (80%/70%) passes in all three modules; the core unit tests (§12.1) and each starter's integration tests (§12.2) all pass, with no static-state test pollution (§2.4) | Kickoff resolution |
| 11 | `groupId` matches constitution §2.11 across all three modules; artifactIds are exactly `log-control-core`, `log-control-spring-boot-starter`, `log-control-spring-boot4-starter`; property prefix is `logcontrol.*` on both starters | Constitution §2.11 |
| 12 | `log-control-core` has zero compile-time dependency on Spring, Jakarta EE, Jackson, or Micrometer | Constitution §2.12 |
| 13 | A service can depend on exactly one starter and never needs the other on its classpath at the same time | Constitution §2.12 |
| 14 | Both starters are released, tested, and available at v1 launch — neither is a stub or placeholder | Constitution §2.12 (reaffirmed 2026-09-01) |

---

## 16. Out of scope (v1) — reminder

Unchanged from constitution §3: not a general logging framework replacement, no log
shipping/aggregation infrastructure, no enforcement of situation-code *semantic* correctness
beyond uniqueness/pattern, no backends other than Logback, no web stacks other than MVC/WebFlux.

---

## Appendix: consuming-service usage example

```java
// 1. Dependency (pom.xml) — pick the starter matching this service's Spring Boot generation:
//
// On Spring Boot 3.5.x:
// <dependency>
//   <groupId>com.llizzard.logcontrol</groupId>
//   <artifactId>log-control-spring-boot-starter</artifactId>
//   <version>1.0.0</version>
// </dependency>
//
// On Spring Boot 4.1.x (currently 4.1.1 in production):
// <dependency>
//   <groupId>com.llizzard.logcontrol</groupId>
//   <artifactId>log-control-spring-boot4-starter</artifactId>
//   <version>1.0.0</version>
// </dependency>
//
// (log-control-core arrives transitively either way — never declared directly.)

// 2. logback-spring.xml — identical regardless of starter chosen
// <include resource="META-INF/logcontrol/logback-include.xml"/>

// 3. This service's situation-code catalog — identical regardless of starter chosen
public enum OrderSituationCode implements LogSituationCode {
    ORDER_CREATED("ORDR-FLW-001", "Order successfully created"),
    ORDER_NOT_FOUND("ORDR-FLW-002", "Requested order id does not exist"),
    PAYMENT_GATEWAY_TIMEOUT("ORDR-EXT-014", "Timed out calling the payment gateway");
    // constructor/getters ...
}

// 4. Usage in a controller/service — identical regardless of starter chosen
private static final ContextualLogger log = ContextualLoggerFactory.getLogger(OrderService.class);

public Order getOrder(String id) {
    Order order = repository.findById(id);
    if (order == null) {
        log.warn(OrderSituationCode.ORDER_NOT_FOUND, "No order found for id {}", id);
        throw new OrderNotFoundException(id);
    }
    log.info(OrderSituationCode.ORDER_CREATED, "Order {} retrieved", id);
    return order;
}
```

Resulting text-mode log line (identical output shape on either starter):

```
2026-09-01 10:14:02.113 WARN  [http-nio-8080-exec-3] c.a.OrderService - method=GET endpoint=/orders/{id} code=ORDR-FLW-002 - No order found for id 8842
```
