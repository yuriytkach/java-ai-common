# Quarkus Framework Rules

Mandatory reading before writing, editing, or refactoring Quarkus application code.

## Dependency Injection

- Use constructor injection with `@RequiredArgsConstructor`.
- Use `@ApplicationScoped` for services unless a narrower or broader scope is explicitly required.

## CDI Interception and Self-Invocation

Quarkus interceptor-based annotations (`@Transactional`, `@Blocking`, `@CacheResult`,
`@WithSession`, custom CDI interceptors, etc.) only fire when the annotated method is invoked
through the CDI proxy — i.e. via a reference to the injected bean held by *another* bean. They
do **not** fire on internal calls within the same bean: `this.method()`, `this::method`, or any
method reference passed to a helper.

This is silent — the code compiles, the test that does not exercise the real transaction boundary
passes, and the missing interception only surfaces in production (partial writes, lost
transactions, broken caching).

```java
// WRONG — onEvent's method reference bypasses the proxy; @Transactional NEVER fires
@ApplicationScoped
class OrderHandler {
  void onEvent(final List<Order> batch) {
    handleBatchWithAck(batch, this::handle);   // <-- direct this::handle, no proxy
  }

  @Transactional
  void handle(final Order order) { … }
}

// CORRECT — annotate on a method called through CDI, OR move the work to a separate bean
@ApplicationScoped
class OrderHandler {
  private final OrderWriter writer; // injected, so CDI proxy is in front of it

  void onEvent(final List<Order> batch) {
    handleBatchWithAck(batch, writer::write);  // <-- via injected bean → proxy in path
  }
}

@ApplicationScoped
class OrderWriter {
  @Transactional
  void write(final Order order) { … }
}
```

Whenever you add an interceptor-driven annotation, trace every caller. If any caller is in the
same bean (`this`-based or a method reference to `this`), either move the annotated method to a
separate bean or restructure so the boundary lives at the proxy.

## Multi-Tenant Hibernate and Request-Scoped TenantResolver

Skip this section if the project does not use Hibernate multi-tenancy (no `TenantResolver`
implementation, no `quarkus.hibernate-orm.multitenant` configuration). When it does apply, the
rule is non-negotiable.

In multi-tenant projects the `TenantResolver` is almost always `@RequestScoped` — it reads the
current tenant from a per-request holder (header, JWT claim, MDC, a `ThreadLocal`-backed context).
Hibernate consults it every time a `Session` is opened: an HTTP request has an active CDI
request context automatically, so the resolver is reachable and everything just works.

**Non-HTTP entry points do not get a request context for free.** Kafka consumers, `@Scheduled`
methods, `CompletableFuture` tasks dispatched via `runAsync`/`supplyAsync` (or a `ManagedExecutor`),
MicroProfile `@Asynchronous` methods, gRPC handlers, message-driven beans — none of them
auto-activate a CDI request scope. The first DB call inside such a flow opens a Hibernate session,
the multi-tenant `SessionFactory` asks for the tenant id, the request-scoped resolver cannot be
resolved, and Hibernate fails with:

```
SessionFactory configured for multi-tenancy, but no tenant identifier specified
```

The failure is silent in tests that mock the repository or run on a single schema. It only fires
in environments where multi-tenancy is actually wired up — typically dev/staging/prod.

Fix: annotate the entry-point method with `@ActivateRequestContext`. Even if the tenant value
itself lives in a `ThreadLocal` (e.g. `TenantContext.setTenant(...)` set by an upstream
header reader), the resolver bean itself still needs an active request scope to be looked up.

```java
// WRONG — no request scope active; @RequestScoped MultiTenantResolver is unreachable
@ApplicationScoped
class UserActivityEventHandler {
  void handle(final Set<UserActivityEvent> events) {
    repository.upsertBatch(toRows(events));   // @Transactional → opens Session → boom
  }
}

// CORRECT — @ActivateRequestContext activates the scope for the duration of the call
@ApplicationScoped
class UserActivityEventHandler {
  @ActivateRequestContext
  void handle(final Set<UserActivityEvent> events) {
    repository.upsertBatch(toRows(events));
  }
}
```

This interacts tightly with the CDI self-invocation rule above: `@ActivateRequestContext` is an
interceptor like `@Transactional`, so it only fires when the annotated method is reached through
the CDI proxy. A Kafka consumer that calls `super.handleBatchWithAck(..., this::handle)` bypasses
the proxy and the annotation is dead. Move the annotated method to a separate bean (the standard
"consumer delegates to handler bean" pattern) so the call path goes through an injected reference.

When you add a new non-HTTP entry point in a multi-tenant project, the checklist is:
1. Does the flow eventually open a Hibernate `Session` (any `@Transactional`, repository call, or
   `EntityManager` access)?
2. Is the `TenantResolver` `@RequestScoped`?
3. If both yes, annotate the entry-point method with `@ActivateRequestContext` and verify the
   method is called through a CDI proxy (i.e. via an injected bean reference, not `this::`).

## Vault-backed Configuration

- **Never bind a Vault-sourced `Map<String, ?>` via `@ConfigMapping`.** The Quarkus Vault
  `VaultConfigSource.getPropertyNames()` returns an empty set, so SmallRye Config — which
  builds Map properties by enumerating property names across sources — silently produces
  an empty map whenever Vault is the only source for that prefix. Property-file / dev-services
  seeds in tests *do* enumerate, so tests pass; Vault-only environments (usually staging and
  prod) get an always-empty map and break in production. Use one of:
  - `Config.getOptionalValue("prefix." + key, T.class)` for per-key lookup (the `getValue`
    path on `VaultConfigSource` works correctly), or
  - `VaultKVSecretEngine` for a programmatic read when the full map is genuinely needed at
    startup.

## Mapping

- Use MapStruct for DTO conversion.
- Use `@Mapper(componentModel = "jakarta")` for Quarkus-managed mappers.

## REST Clients

- Use `@RestClient` for external API clients.
- Follow existing project patterns for request/response DTOs, error mapping, and timeouts.

## Error Handling

- Prefer RFC 7807 Problem Details when the project uses the Quarkus problem-details stack.
- Keep exception-to-response mapping aligned with existing Quarkus resources and exception mappers.
