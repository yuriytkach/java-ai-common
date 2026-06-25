# Spring Boot Testing Rules

Mandatory reading before writing or modifying Spring Boot test classes.

## Unit vs. Integration Tests

- The framework-bootstrapping annotation in Spring Boot is `@SpringBootTest`.
- Unit tests MUST NOT use `@SpringBootTest` or any other context-loading annotation.
- Use `@SpringBootTest` for full integration tests.
- Prefer slice tests such as `@WebMvcTest` or `@DataJpaTest` when a full context is unnecessary.

## Reuse Existing Test Infrastructure Before Adding Parallel Wiring

Before adding a new test resource or test container, **survey what's already there**. A second
copy of the same wiring forces the test harness to start both, and the two copies drift apart
silently.

Before adding any of the following, grep / read the existing project state to confirm it
isn't already provided:

- **A test resource for a service (DB, Redis, Kafka, Vault, S3)** — check for an existing
  `@Testcontainers` setup with a `@Container` field, a shared `AbstractContainerIT` base class,
  or a `@TestConfiguration` that exposes the service bean.
- **A new test container** — check for an existing `@Container` field in a base IT class or a
  `@DynamicPropertySource` that already wires the same image.

The mistake shape: you add a `@Container static GenericContainer<> redis = ...` to a new IT.
A shared `AbstractRedisIT` base class already starts the same container and exposes it to every
subclass via `@DynamicPropertySource`; the new IT just needed to extend it. The two containers now
race for the same Docker host port between test classes.
