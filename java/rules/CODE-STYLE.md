# Code Style Rules

Mandatory reading before writing, editing, or refactoring any Java code.

## Formatting

- Indentation: 2 spaces (NOT 4).
- Line Length: Soft limit 100-120 characters.
- Braces: Egyptian style (opening brace at end of line).
- Imports: NO wildcard imports. Order: Standard Java, Jakarta/Javax, Org/Third-party, Com (Project),
  Lombok.
- Fully qualified class names: NEVER write fully qualified class names inline. Always add an
  `import` and use the short name. This applies to every reference - method parameters, return
  types, field types, local variables, generics, annotations, exception types, cast targets, and
  `instanceof` checks. The ONLY exception is a genuine naming collision where two imported types
  share a short name; in that case import one and qualify only the second. Do NOT use FQNs to "save
  time" on long tasks or to avoid scrolling to the import block.

```java
// WRONG - fully qualified name inline
public java.util.List<com.example.orders.Order> findOrders(final java.time.LocalDate date) {
  final java.util.Map<String, java.lang.Integer> counts = new java.util.HashMap<>();
  ...
}

// CORRECT - imports at top, short names in code
import java.time.LocalDate;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import com.example.orders.Order;

public List<Order> findOrders(final LocalDate date) {
  final Map<String, Integer> counts = new HashMap<>();
  ...
}
```

## Java Language Features

- Java 21. Use final var for local variables. Prefer final for parameters and fields.
- Lombok: Use @Data/@Value for DTOs, @RequiredArgsConstructor for injection, @Slf4j for logging.
  Do NOT use `@SneakyThrows` in production code. In production code, catch exceptions explicitly or
  declare them in the method signature. `@SneakyThrows` is allowed in tests only.
- Streams: Use Java Streams API and `one.util.streamex.StreamEx` for collection processing.
- Async: Use CompletableFuture as the preferred default for async composition when async work is justified.
  Do NOT introduce parallelism without a concrete need. ExecutorService and other lower-level primitives
  are valid when the use case requires explicit control.
- Nullability: Mark nullable fields and parameters with `@Nullable` from the `javax.annotation`
  package. `@Nullable` is a **contract claim**, not defensive padding. Before annotating a parameter
  / field / record component `@Nullable`, check the call sites you can see. If they all pass
  non-null values, the parameter is not nullable — declaring it so weakens the type signal callers
  rely on, forces the method body to write dead defensive null-guards, and tells future maintainers
  that null is a meaningful state when it isn't. Default to non-null. Mark `@Nullable` only when
  null is a documented, meaningful state in the domain. For unbounded public APIs where callers
  aren't all visible, still default to non-null and let the type signal carry the contract.

```java
// WRONG — all callers pass clock.now(), which is never null. The @Nullable + null-guard are dead.
public void recordCall(@Nullable final Instant ts) {
  if (ts == null) {
    return;
  }
  pendingCall.set(ts);
}

// CORRECT — parameter is non-null by contract; callers cannot pass null without a compile-time signal.
public void recordCall(final Instant ts) {
  pendingCall.set(ts);
}
```

- `Optional`: NEVER use `Optional` as a method parameter or field of class/record. It is intended only
  as a return type. Use `@Nullable` and overloads instead:

```java
// WRONG
public long countCoRegistrations(final Optional<YearMonth> month) { ... }

// CORRECT - nullable parameter
public long countCoRegistrations(@Nullable final YearMonth month) { ... }

// CORRECT - overload for the absent case
public long countCoRegistrations() { ... }
public long countCoRegistrations(final YearMonth month) { ... }
```

- **Stream idioms**: Prefer Java 21 functional sugar over verbose equivalents:
  - Flat-map an `Optional` result using `.flatMap(x -> toOptional(x).stream())`. Never use
    `.map(x -> optional.orElse(null))` followed by a null filter (`.filter(Objects::nonNull)` or
    StreamEx `.nonNull()`).
  - Negate a predicate with `Predicate.not(...)` (static-imported as `not(...)`). Never write a
    lambda negation `e -> !e.method()` when a method reference exists.

```java
// WRONG
.map(session -> toResponse(session).orElse(null)).nonNull()
.filter(e -> !e.isActive())

// CORRECT
.flatMap(session -> toResponse(session).stream())
.filter(not(Entity::isActive))
```

## Naming Conventions

- Classes: PascalCase, e.g. CompanyService
- Methods/Variables: camelCase, e.g. getCompanyById
- Constants: UPPER_SNAKE_CASE, e.g. MAX_RETRY_COUNT
- Exception variables: ALWAYS name them `ex`, never `e`.
- **Do not use Java reserved or restricted identifiers** (`record`, `yield`, `sealed`, `permits`,
  `var`, `_`, plus the long-standing reserved words) as method, field, parameter, or local-variable
  names. Sonar S6213 flags `record` specifically; IDE refactors silently break when a method
  shadows a keyword. Pick a verb that describes the action — `capture`, `register`, `apply`.

## Configuration Properties

- **Duration fields**: Always use `java.time.Duration` as the field type for any property that
  represents a time interval. Never use `int`/`long` with a unit suffix like `timeoutInMs`,
  `delaySeconds`, or `intervalSec`. Spring Boot and Quarkus both bind duration strings
  (e.g. `2000ms`, `5s`, `1m`) automatically, so no manual conversion is needed at the call site.

```java
// WRONG
public record AppProperties(long retryDelayInMs, int maxWaitSeconds) {}

// CORRECT
public record AppProperties(Duration retryDelay, Duration maxWait) {}
```

## Error Handling

- Define custom exceptions extending RuntimeException, or use the project's Problem Details convention.
- Never swallow exceptions. Log and rethrow, or handle specific cases explicitly.

### Asynchronous failure paths

Any method that returns a future-like type (`CompletableFuture`, `CompletionStage`, Mutiny `Uni`,
RxJava/Reactor publishers) can fail *asynchronously* — long after the call site returned. A
synchronous `try/catch` around the call only covers the throw-from-`invoke` case; it does NOT cover
exceptional completion of the returned future. If you wrote cleanup, throttle release, retry
bookkeeping, or any other side effect in that `catch`, you owe the same side effect on the
exceptional-completion path.

```java
// WRONG — async nack / Uni failure leaks past the catch
try {
  sendAsync(value)
    .whenComplete((ignored, err) -> { /* nothing — failures slip through */ });
} catch (final Exception ex) {
  releaseResource();
  log.error("send failed: {}", ex.getMessage(), ex);
}

// CORRECT — both sync throws AND async exceptional completions route through the same cleanup
try {
  sendAsync(value)
    .whenComplete((ignored, asyncEx) -> {
      if (asyncEx != null) {
        releaseAndLog(asyncEx);
      }
    });
} catch (final Exception ex) {
  releaseAndLog(ex);
}
```

Whenever you call a future-returning method, explicitly answer: *what happens if it completes
exceptionally?* Apply the same cleanup / fail-open / logging semantics as the sync-throw path.

## Logging

Add observability proactively, not as an afterthought. The policy below applies to **all** new
code — including code that does not yet contain any log statement:

- Log at `INFO` for important calls and decisions: starting a significant operation, a branching
  decision with business impact, completion of an external call, or a notable state transition.
- Log at `WARN` for recoverable unexpected states; `ERROR` only for failures that require attention.
- Do NOT log every trivial step. Aim for one or two `INFO` lines per meaningful unit of work —
  enough signal to reconstruct a request's journey without noise.

Exact exception-logging format, MDC key conventions, and MDC propagation across async boundaries
live in `LOGGING.md`. Read that file before adding or changing any log statement, MDC key, or
async dispatch that needs the caller's logging context.
