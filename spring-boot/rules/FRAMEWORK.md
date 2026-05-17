# Spring Boot Framework Rules

Mandatory reading before writing, editing, or refactoring Spring Boot application code.

## Dependency Injection

- Use constructor injection with `@RequiredArgsConstructor`.
- Use `@Service`, `@Component`, and `@Repository` according to the component's responsibility.
- Avoid field injection unless a framework constraint leaves no clean alternative.

## AOP Interception and Self-Invocation

Spring's proxy-based AOP annotations (`@Transactional`, `@Async`, `@Cacheable`, `@CacheEvict`,
`@Retryable`, `@Validated` on methods, custom `@Aspect` advice, etc.) only fire when the
annotated method is invoked through the Spring-managed proxy — i.e. via a reference to the
injected bean held by *another* bean. They do **not** fire on internal calls within the same
bean: `this.method()`, `this::method`, or any method reference passed to a helper.

This is silent — the code compiles, a unit test that mocks the dependency boundary passes, and
the missing advice only surfaces in production (partial writes, swallowed retries, broken
async dispatch, lost cache evictions).

```java
// WRONG — onBatch's method reference bypasses the proxy; @Transactional NEVER fires
@Service
@RequiredArgsConstructor
class OrderHandler {
  void onBatch(final List<Order> batch) {
    batch.forEach(this::save);   // <-- this::save → no proxy
  }

  @Transactional
  void save(final Order order) { … }
}

// CORRECT — annotate on a method called through Spring DI, OR extract to a separate bean
@Service
@RequiredArgsConstructor
class OrderHandler {
  private final OrderWriter writer;  // injected → proxy in front

  void onBatch(final List<Order> batch) {
    batch.forEach(writer::save);     // <-- via injected bean → proxy in path
  }
}

@Service
class OrderWriter {
  @Transactional
  public void save(final Order order) { … }
}
```

Whenever you add an AOP-driven annotation, trace every caller. If any caller is in the same bean
(`this`-based or a method reference to `this`), either move the annotated method to a separate
bean or restructure so the boundary lives at the proxy.

## Mapping

- Use MapStruct for DTO conversion.
- Use `@Mapper(componentModel = "spring")` for Spring-managed mappers.

## Configuration

- Use `@ConfigurationProperties` for grouped typed configuration.
- Prefer constructor-bound or otherwise immutable configuration objects where the project supports them.

## REST Clients

- In Spring Boot 3.x projects, prefer `@FeignClient` for new declarative HTTP clients when OpenFeign
  is already used by the project or the task explicitly introduces it.
- Do NOT rewrite existing `RestTemplate`, Apache HttpClient, or similar legacy client code unless the
  task explicitly includes such a migration.
- In Spring Boot 2.x projects, preserve the existing client style unless the project already uses
  `@FeignClient` or there is a clear explicit reason to introduce it.
- In Spring Boot 3.x projects, do NOT introduce new `RestTemplate` clients unless matching an established
  project pattern or explicitly requested.

## Error Handling

- Prefer Spring's `ProblemDetail` model for API error responses.
- Keep exception-to-response mapping aligned with the project's `@ControllerAdvice`, exception handlers,
  and existing API error contract.
