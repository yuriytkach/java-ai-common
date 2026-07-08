# Quarkus Testing Rules

Mandatory reading before writing or modifying Quarkus test classes.

## Unit vs. Integration Tests

- The framework-bootstrapping annotation in Quarkus is `@QuarkusTest`.
- Do NOT use `@QuarkusTest` for unit tests.
- Use `@QuarkusTest` only for integration tests in the `quarkusIntTest` source set.

## Reuse Existing Test Infrastructure Before Adding Parallel Wiring

Before adding a new test resource or test container, **survey what's already there**. A second
copy of the same wiring forces the test harness to start both, and the two copies drift apart
silently.

Before adding any of the following, grep / read the existing project state to confirm it
isn't already provided:

- **A `@QuarkusTestResourceLifecycleManager` for a service (DB, Redis, Kafka, Vault, S3)** —
  check for an existing one in `src/integrationTest/.../*TestResource.java` and for active
  `quarkus.<service>.devservices.enabled=true` in `application.yml`.
- **A new test container** — check DevServices first, then existing `@QuarkusTestResource`
  annotations on neighbouring ITs.

The mistake shape: you write a `RedisTestResource extends QuarkusTestResourceLifecycleManager`
that spins up a `GenericContainer<>("redis:7-alpine")`. Quarkus DevServices for Redis was already
enabled in the `%test` profile and provisions an equivalent container; the IT just needed to
inject the existing Redis client. ~50 lines deleted on the follow-up commit.
