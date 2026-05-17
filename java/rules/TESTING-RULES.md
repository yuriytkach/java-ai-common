# Testing Rules

Mandatory reading before writing or modifying any test class.

## Frameworks

- JUnit 5: `@Test`, `@ParameterizedTest`, `@BeforeEach`.
- Mockito: `@ExtendWith(MockitoExtension.class)`, `@Mock`, `@InjectMocks`.
- AssertJ: Preferred for all assertions.
- TestContainers: For integration tests involving DB, Kafka, Vault, or other external infrastructure.

## Unit vs. Integration Tests

- Unit tests mirror the package structure of the class under test.
- Integration tests are named with the `IT` suffix.
- When changing DB code, SQL queries, REST client code, Kafka code, OpenSearch code, Redis code,
  or similar infrastructure-facing integration code, integration tests MUST be added or updated.
- Do NOT rely on unit tests alone for those changes. The integration behavior must be verified against
  the actual framework wiring and infrastructure boundary.

### Mocking the infrastructure boundary — strict prohibition

For an infrastructure-coupled class (repository with custom SQL, Kafka publisher/consumer,
cache-backed service where TTL / eviction / atomicity is part of the behavior, REST client where
serialization or headers matter), an integration test is **the** verification — not an additional
layer on top of a mocked unit test. A mocked unit test for such a class is not a valid substitute
for the IT, even when it passes.

The mock you reach for replaces exactly the part the behavior depends on. The mocked test then
"passes" against semantics that no real component provides — wrong SQL still serialises, wrong
headers still appear, an "expired" cache entry never actually expires. The IT catches all of these;
the mocked unit test cannot.

**MUST NOT** stand alone as the primary verification when the behavior depends on the boundary
they represent:

- `EntityManager`, `Session`, `JdbcTemplate`, `NamedParameterJdbcTemplate`, `JpaRepository` /
  Panache repository APIs.
- `Emitter<T>` (Quarkus Reactive Messaging), `KafkaProducer`, `KafkaConsumer`, Spring
  `KafkaTemplate`, `@KafkaListener` containers.
- `io.quarkus.cache.Cache` / `CaffeineCache`, Spring `Cache` / `CacheManager`, `RedisClient`,
  Redisson clients, when TTL / eviction / atomic claim is part of the behavior under test.
- `RestClient` / `WebClient` / OpenFeign clients when transport details (headers, request body
  serialization, retry policy) are part of the behavior under test.
- `DataSource` / connection-pool primitives when transaction boundaries are part of the behavior.

Mocked unit tests on infrastructure-coupled classes ARE acceptable, but only for **pure behavioral
logic that is independent of the infrastructure semantics** — e.g. argument validation, branching
on inputs, delegation to another collaborator, transformation of inputs before the boundary call.
The moment a test asserts something the mock cannot replicate (TTL expiry, real SQL execution,
real header serialization), the test belongs in the IT layer.

```java
// WRONG — mocked EntityManager "test" passes against any SQL string, including broken SQL
@ExtendWith(MockitoExtension.class)
class OrderRepositoryTest {
  @Mock EntityManager em;
  @InjectMocks OrderRepository tested;

  @Test void shouldFindByStatus() {
    when(em.createQuery(any(String.class), eq(Order.class))).thenReturn(typedQuery);
    when(typedQuery.getResultList()).thenReturn(List.of(order));

    assertThat(tested.findByStatus(ACTIVE)).containsExactly(order);
  }
}

// CORRECT (Quarkus) — repository behavior verified against a real Postgres via DBRider
@QuarkusTest
@DBRider
class OrderRepositoryIT {
  @Inject OrderRepository tested;

  @Test
  @DataSet("OrderRepositoryIT/orders.yml")
  void shouldFindByStatus() {
    assertThat(tested.findByStatus(ACTIVE)).extracting(Order::id).containsExactly(1L, 2L);
  }
}

// CORRECT (Spring) — repository behavior verified against a real Postgres via Testcontainers
@DataJpaTest
@Testcontainers
@AutoConfigureTestDatabase(replace = Replace.NONE)
class OrderRepositoryIT {
  @Container
  static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");

  @Autowired OrderRepository tested;

  @Test
  @Sql("/OrderRepositoryIT/orders.sql")
  void shouldFindByStatus() {
    assertThat(tested.findByStatus(ACTIVE)).extracting(Order::id).containsExactly(1L, 2L);
  }
}
```

```java
// WRONG — mocked Cache "TTL test" cannot replicate Caffeine's eviction semantics
@Test void shouldEvictAfterTtl() {
  when(cache.getIfPresent(any())).thenReturn(value, null); // fake "expiry" via stubbed sequence
  // … passes against a contract no real Caffeine cache implements.
}

// CORRECT (Quarkus) — exercise the real cache via @CacheName injection in an IT
@QuarkusTest
class ThrottleServiceIT {
  @Inject @CacheName("throttle") Cache throttle;
  @Inject ThrottleService tested;

  @BeforeEach void clear() { throttle.invalidateAll().await().indefinitely(); }

  @Test void shouldSuppressDuplicateWithinTtl() { … real cache, real semantics … }
}

// CORRECT (Spring) — exercise the real cache via CacheManager in an IT
@SpringBootTest
class ThrottleServiceIT {
  @Autowired CacheManager cacheManager;
  @Autowired ThrottleService tested;

  @BeforeEach void clear() { cacheManager.getCache("throttle").clear(); }

  @Test void shouldSuppressDuplicateWithinTtl() { … real cache, real semantics … }
}
```

If a class is split such that the pure-logic helper lives separately from the infrastructure
adapter (a `Service` that delegates to a `Repository`, or a `*Aggregator` separate from the
`*Consumer`), the helper is a fine target for a mocked unit test, and the adapter is the target
for the IT. That split is preferred over piling both responsibilities into one class and then
arguing about which test layer to use.

## Testing Configuration Objects

Both Quarkus (`@ConfigMapping`) and Spring (`@ConfigurationProperties`) bind grouped configuration
to typed interfaces or classes that often contain nested interfaces or records for sub-groups.

**Prefer binding real configuration values over mocking the config object.** A test that binds
real property values exercises the actual mapping, the actual nested structure, and the actual
type coercion the framework will use in production. Excessive mocking of nested configuration
objects hides binding mistakes that would have shown up in a real test profile.

When you must mock the configuration object (e.g. a focused unit test where binding the full
profile is overkill), annotate the mock with `@Mock(answer = Answers.RETURNS_DEEP_STUBS)` so
nested accessors can be stubbed in a single `when(...)` chain without creating a separate mock
for each sub-interface.

```java
// CORRECT
@Mock(answer = Answers.RETURNS_DEEP_STUBS)
AppProperties properties;

when(properties.s3().bucketName()).thenReturn("my-bucket");

// WRONG - unnecessary intermediate mock for each nested interface
@Mock S3Settings s3;
when(properties.s3()).thenReturn(s3);
when(s3.bucketName()).thenReturn("my-bucket");
```

Use `RETURNS_DEEP_STUBS` only for configuration/properties interfaces, not for service or
repository mocks where deep stubbing would hide missing explicit stubs.

## DBRider and @Nested - Strict Prohibition

**NEVER use `@Nested` inner classes in DBRider-based integration tests.**

DBRider interceptors do not reliably fire for methods inside `@Nested` classes, so dataset setup and
teardown can be silently skipped, leading to corrupt test state. When grouping is needed, create a
separate top-level IT class instead:

```java
// WRONG - DBRider annotations will not be applied inside @Nested
class OrderRepositoryIT {
  @Nested
  class FindByStatus { ... }
}

// CORRECT - split into top-level classes
class OrderRepositoryFindByStatusIT { ... }
class OrderRepositoryCountByMonthIT { ... }
```

This rule applies to any test class that uses DBRider, regardless of whether the `@Nested` class
has its own dataset annotations.

## Test Organisation with @Nested

Use `@Nested` inner classes to group related test cases when a test class covers multiple scenarios
or contains many test methods. For tiny test classes, `@Nested` is optional.

### Naming

- Test classes: `UserServiceTest`, `UserRepositoryIT` (suffix `Test` or `IT`).
- `@Nested` classes: describe the method or condition - `GetCompanyById`, `WhenInputIsInvalid`, `EdgeCases`.
- Test methods: `shouldDoSmthWhenSmth` - e.g. `shouldReturnCompanyWhenFound()`.

### Canonical Structure

```java
class CompanyServiceTest {
  @Nested
  class GetCompanyById {
    @Test void shouldReturnCompanyWhenFound() { }
    @Test void shouldThrowExceptionWhenIdIsNull() { }
  }

  @Nested
  class CreateCompany {
    @Nested class WhenInputIsValid {
      @Test void shouldPersistCompanyToDatabase() { }
    }

    @Nested class WhenInputIsInvalid {
      @ParameterizedTest
      @NullAndEmptySource
      void shouldThrowValidationExceptionWhenNameIsBlank(final String name) { }
    }
  }
}
```

## Parameterized Tests

Use `@ParameterizedTest` whenever the same behaviour needs to be verified across multiple input
values. Do NOT write a separate `@Test` method per value.

This is especially important for validation: a single parameterized test replaces a proliferation
of near-identical methods like `shouldThrowWhenNameIsNull`, `shouldThrowWhenNameIsEmpty`, etc.

Choose the most concise source annotation for the situation:

| Situation | Annotation |
|---|---|
| Null and/or empty strings | `@NullAndEmptySource`, `@NullSource`, `@EmptySource` |
| Fixed set of primitives or strings | `@ValueSource` |
| Multiple parameters per case | `@CsvSource` |
| Complex objects or many cases | `@MethodSource` |

```java
// Validation - multiple invalid values collapsed into one test
@ParameterizedTest
@NullAndEmptySource
@ValueSource(strings = {" ", "\t"})
void shouldThrowWhenCompanyNameIsBlank(final String name) {
  assertThatThrownBy(() -> service.createCompany(name))
    .isInstanceOf(BadRequestProblem.class);
}

// Multiple parameters per case
@ParameterizedTest
@CsvSource({
  "PENDING, 0",
  "ACTIVE,  5",
  "CLOSED,  5"
})
void shouldReturnCorrectCountForStatus(final String status, final int expected) { ... }
```

## Coverage of New Branches

Whenever you introduce a new branching construct — `try`/`catch`, `if`/`else`, ternary, switch arm,
`whenComplete` lambda, defensive null check — add a test that exercises **each arm** in the same
commit. Coverage is not a follow-up step.

The arms most often missed are:

- The **defensive** `catch` block (the "this should never happen, but fail-open" branch).
- The **success** arm of a `whenComplete` (`asyncEx == null`) when the test only checks the
  failure path.
- The **else** branch of an `if` that guards a rare condition.
- The **null/empty** input branch of a method whose happy path is the test's focus.

Letting a coverage tool (Sonar, JaCoCo) flag these later forces a second review round and tends
to result in coverage-only tests bolted on after the fact. Add the test up front instead.

```java
// WRONG — only the throw path is tested; the success path of whenComplete is never asserted
@Test void shouldReleaseResourceOnAsyncFailure() {
  tested.send(value);
  capturedMessage.nack(new RuntimeException());
  assertThat(resource.isReleased()).isTrue();
}

// CORRECT — both arms covered in the same commit
@Test void shouldReleaseResourceOnAsyncFailure() { … nack path … }
@Test void shouldKeepResourceOnAsyncSuccess() { … ack path … }
```

## Scope Test-Only Affordances Narrowly

Methods, constructors, or accessors that exist **only** to enable testing — `resetForTest()`,
`forTesting(...)`, exposing an internal counter, a setter for an otherwise-immutable field — should
be scoped as narrowly as the call sites permit. The hierarchy:

1. Don't add the affordance at all — restructure the test to use the real API surface or move the
   test into the same package as the production class so the regular access modifier is enough.
2. Package-private, with same-package tests calling it.
3. Public, only when a deliberate API decision says the operational hook belongs on the surface.

Promoting a test-only method to `public` because a test in a different package needs it is almost
always a sign that the cross-package test has fragile isolation; fix that test first, then keep
the affordance package-private. A `public resetForTest()` invites production callers, silently
widens the supported API surface, and signals to future maintainers that the method is a real
operational hook.

```java
// WRONG — public so an IT in com.example.service can reach into the recorder package
@ApplicationScoped                 // or @Service / @Component depending on the stack
public class StatusRecorder {
  ...
  public void resetForTest() { ... }   // any production code can now call this
}

// CORRECT — package-private; only tests in com.example.health.recorder can call it
@ApplicationScoped                 // or @Service / @Component depending on the stack
public class StatusRecorder {
  ...
  /** Test-only fixture cleanup. Production code MUST NOT call this. */
  void resetForTest() { ... }
}
```

If a Kotlin / different-package test genuinely needs to call the affordance, the right escape
hatches (in order of preference) are: (a) make the test class same-package, (b) introduce a
narrow `@VisibleForTesting`-equivalent annotation that lints other callers, (c) accept the public
visibility AND document on the method why production code must not call it. Going straight to
"make it public" without that audit trail is the wrong default.

## Wall-clock Timing in Tests

Do NOT rely on wall-clock delays in the single-digit-millisecond range to gate test behaviour.
JIT compilation, GC pauses, container scheduling, and CI-runner contention routinely push two
back-to-back Java method calls past 1 ms apart — a test that "two `publish()` calls happen within
the 1 ms TTL" passes locally and fails on Jenkins.

When a test needs deterministic cache / throttle / TTL state, control it **explicitly**:

- Invalidate caches in `@BeforeEach` (e.g. `cache.invalidateAll().await().indefinitely()`).
- Use the cache's `keySet()` / size to observe state directly rather than inferring it from a
  side effect that depends on timing.
- Keep configured TTLs at realistic production values; reset state between tests instead of
  shrinking the TTL to a number that's racing the JVM.

```java
// WRONG — relies on PT0.001S TTL being longer than two consecutive Java calls
tested.publish(event);
tested.publish(event);
assertThat(messages).hasSize(1); // flaky on CI

// CORRECT — production-sized TTL, explicit per-test reset
@BeforeEach void resetCache() { throttleCache.invalidateAll().await().indefinitely(); }

@Test void shouldThrottleSecondPublishWithinTtl() {
  tested.publish(event);
  tested.publish(event);
  WAIT.untilAsserted(() -> assertThat(messages).hasSize(1));
  // Optional: pause and re-assert to catch a leaked second message.
}
```

If you must wait on an external system you cannot control (CI job, downstream service),
use Awaitility's `await()` with conservative `pollDelay` / `atMost` budgets, not raw
`Thread.sleep` of tens of milliseconds.

## Stubs vs. Verification

`when(...)` stubs configure behaviour — they are NOT assertions.
Always use `any()` (or `any(Type.class)`) matchers in `when(...)` stubs.
Never use `eq()` or raw argument values inside `when(...)` — they create the
illusion of verification while actually asserting nothing: if the code never
calls the method, or calls it with wrong arguments but the stub condition
simply doesn't match, the test still passes silently.

To assert that a method was called with specific arguments, use `verify(...)`.

```java
// WRONG — eq() in when() is not an assertion
when(service.findByCode(eq(offerCode), eq(prefix), any())).thenReturn(Stream.of());

// CORRECT — stub accepts any args, verify asserts the actual call
when(service.findByCode(any(), any(), any())).thenReturn(Stream.of());
// ...execute code under test...
verify(service).findByCode(offerCode, prefix, expectedAuth);
```

## Closed-Set Matchers in Stubs — Avoid `EnumSet.allOf` / `.values()` / Wildcards

Do NOT use `EnumSet.allOf(MyEnum.class)`, `Arrays.asList(MyEnum.values())`, or any other
"give me everything in the universe" expression as a Mockito match argument in `when(...)` /
`verify(...)`. Such matchers silently change meaning the moment the underlying enum (or other
closed set) gains a new value — the stub stops matching what the production code now sends, and
the test fails or, worse, silently returns `null` from an unmatched stub and breaks elsewhere.

```java
// WRONG — ALL_METRICS = EnumSet.allOf(Metric.class) is 5 today, 6 tomorrow.
// Adding ANY_ACTIVITY_USERS to Metric silently makes the controller send a 5-element EnumSet
// while the stub still expects 6 — mock returns null, JSON body has nulls, assertion fails at
// runtime (not compile time, because EnumSet sizes are runtime data).
private static final Set<Metric> ALL_METRICS = EnumSet.allOf(Metric.class);

when(service.getSplit(any(), any(), any(), any(), anyBoolean(), any(), ALL_METRICS))
  .thenReturn(response);

// CORRECT — pin the EnumSet to the values the test cares about, by name.
when(service.getSplit(any(), any(), any(), any(), anyBoolean(), any(),
    EnumSet.of(Metric.USERS, Metric.CLIPS, Metric.REDEMPTIONS,
      Metric.UNIQUE_CLIPPERS, Metric.UNIQUE_REDEEMERS)))
  .thenReturn(response);
```

When you extend an enum / sealed type / closed value set, also sweep the test sources for every
`allOf` / `.values()` / `Arrays.asList(X.values())` / "all of them" wildcard reference and convert
each to an explicit enumeration of the values the test actually depends on. The grep:

```
grep -rn "EnumSet.allOf\|.values()\|all.*Class" src/test src/integrationTest
```

…then triage each hit. Tests that genuinely need "all current values" (e.g. a contract test that
the public API surface includes every enum value) are the rare exception; everything else should
be explicit.

## Verify with Real Argument Values

In `verify(...)` calls, always pass the exact expected argument values. Matchers like `any()` or
`anyString()` are fine in `when(...)` stubs but MUST NOT appear in `verify(...)` unless the exact
value is genuinely unknowable at test time, in which case add a comment explaining why.

```java
// CORRECT
verify(orderService).createOrder(orderId, customerId);

// WRONG - masks incorrect arguments
verify(orderService).createOrder(any(), any());

// Justified exception - value is generated internally
verify(eventPublisher).publish(any(OrderCreatedEvent.class)); // UUID generated at call site
```

## Asserting Absence of Interactions

Prefer `verifyNoInteractions(mock)` or `verifyNoMoreInteractions(mock)` over
`verify(mock, never()).method(...)`.

`verify(mock, never()).method(arg)` only guards against one specific call signature — if production
code later calls the same method with different arguments, the assertion passes silently.
`verifyNoInteractions` / `verifyNoMoreInteractions` catch any call to any method on the mock.

Use `never()` only when you need to assert that one specific method was not called while other
methods on the same mock legitimately were.

```java
// WRONG - passes silently if called with different arguments
verify(cache, never()).evictCacheEntry(USER_ID, "");

// CORRECT - catches any call to any method on cache
verifyNoInteractions(cache);

// Justified never() - other methods on the same mock are expected to be called
verify(cache).put(key, value);
verify(cache, never()).evictCacheEntry(any(), any());
```

## ArgumentCaptor

Use `ArgumentCaptor` only when the argument cannot be verified by passing a value directly to
`verify(...)`. If you can construct or reference the expected value, verify it directly - a captor
in that situation is noise.

```java
// WRONG - captor is unnecessary, the value is known
@Captor ArgumentCaptor<YearMonth> monthCaptor;
verify(repository).countByMonth(monthCaptor.capture());
assertThat(monthCaptor.getValue()).isEqualTo(yearMonth);

// CORRECT - verify directly
verify(repository).countByMonth(yearMonth);
```

When a captor is genuinely needed (e.g. the argument is an object constructed internally by the
class under test), declare it as a field annotated with `@Captor`. Do NOT create it inline with
`ArgumentCaptor.forClass(...)`.

```java
// CORRECT - OrderRequest is built internally, cannot be reproduced from the test
@Captor ArgumentCaptor<OrderRequest> orderRequestCaptor;

verify(orderService).createOrder(orderRequestCaptor.capture());
assertThat(orderRequestCaptor.getValue().getCustomerId()).isEqualTo(customerId);
```
