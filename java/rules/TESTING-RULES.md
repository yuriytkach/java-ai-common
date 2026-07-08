---
paths:
  - "**/src/test/**"
  - "**/src/integrationTest/**"
  - "**/src/quarkusIntTest/**"
  - "**/*Test.java"
  - "**/*IT.java"
---

# Testing Rules

Mandatory reading before writing or modifying any test class.

## Frameworks

- JUnit 5: `@Test`, `@ParameterizedTest`, `@BeforeEach`.
- Mockito: `@ExtendWith(MockitoExtension.class)`, `@Mock`, `@InjectMocks`.
- AssertJ: preferred for all assertions.
- TestContainers: for integration tests involving DB, Kafka, Vault, or other external infrastructure.

## Unit vs. Integration Tests

- Unit tests MUST NOT bootstrap the framework application context. Framework-bootstrapping
  annotations belong to integration tests in the framework-specific IT source set — see the
  framework-specific `TESTING.md`.
- Unit tests mirror the package structure of the class under test; integration tests carry the
  `IT` suffix.
- When changing DB code, SQL queries, REST client code, Kafka code, OpenSearch code, Redis code,
  or similar infrastructure-facing code, integration tests MUST be added or updated. Unit tests
  alone are not sufficient for those changes.

### Mocking the infrastructure boundary — strict prohibition

For an infrastructure-coupled class (repository with custom SQL, Kafka publisher/consumer,
cache-backed service where TTL/eviction/atomicity is the behavior, REST client where
serialization or headers matter), the integration test IS the verification. A mocked unit test
is not a valid substitute even when it passes: the mock replaces exactly the part the behavior
depends on, so wrong SQL still "works", wrong headers still "appear", and an "expired" cache
entry never actually expires.

Mocks of the following MUST NOT stand alone as primary verification when the behavior depends
on the boundary they represent:

- `EntityManager`, `Session`, `JdbcTemplate`, `NamedParameterJdbcTemplate`, `JpaRepository` /
  Panache repository APIs.
- `Emitter<T>` (Quarkus Reactive Messaging), `KafkaProducer`/`KafkaConsumer`, Spring
  `KafkaTemplate`, `@KafkaListener` containers.
- `io.quarkus.cache.Cache` / `CaffeineCache`, Spring `Cache`/`CacheManager`, Redis clients —
  when TTL / eviction / atomic claim is part of the behavior under test.
- `RestClient` / `WebClient` / OpenFeign clients when transport details (headers, body
  serialization, retry policy) are part of the behavior under test.
- `DataSource` / connection-pool primitives when transaction boundaries are part of the behavior.

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

Mocked unit tests on infrastructure-coupled classes ARE acceptable for pure behavioral logic
independent of the infrastructure semantics — argument validation, branching on inputs,
delegation, input transformation. The moment a test asserts something the mock cannot replicate,
it belongs in the IT layer. Preferred design: split the pure-logic helper from the
infrastructure adapter; unit-test the helper, IT the adapter.

## Testing Configuration Objects

Prefer binding real configuration values over mocking the config object (`@ConfigMapping` /
`@ConfigurationProperties`) — a real binding exercises the actual mapping, nesting, and type
coercion used in production. When a focused unit test justifies mocking the config object, use
deep stubs so nested accessors stub in one chain:

```java
// CORRECT
@Mock(answer = Answers.RETURNS_DEEP_STUBS)
AppProperties properties;

when(properties.s3().bucketName()).thenReturn("my-bucket");

// WRONG — unnecessary intermediate mock for each nested interface
@Mock S3Settings s3;
when(properties.s3()).thenReturn(s3);
when(s3.bucketName()).thenReturn("my-bucket");
```

Use `RETURNS_DEEP_STUBS` only for configuration interfaces — on service or repository mocks it
hides missing explicit stubs.

## DBRider and @Nested — Strict Prohibition

**NEVER use `@Nested` inner classes in DBRider-based integration tests.** DBRider interceptors
do not reliably fire for methods inside `@Nested` classes, so dataset setup/teardown can be
silently skipped, corrupting test state. When grouping is needed, split into top-level IT
classes instead:

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

This applies to any test class using DBRider, whether or not the `@Nested` class has its own
dataset annotations.

## Test Organisation with @Nested

Outside DBRider tests, use `@Nested` classes to group scenarios in larger test classes (optional
for tiny ones):

- Test classes: `UserServiceTest`, `UserRepositoryIT`.
- `@Nested` classes: the method or condition — `GetCompanyById`, `WhenInputIsInvalid`.
- Test methods: `shouldDoSmthWhenSmth` — e.g. `shouldReturnCompanyWhenFound()`.

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

Use `@ParameterizedTest` whenever the same behaviour is verified across multiple input values —
never a separate `@Test` per value. This especially applies to validation, where one
parameterized test replaces a pile of near-identical methods. Pick the most concise source:

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

Every new branching construct — `try`/`catch`, `if`/`else`, ternary, switch arm, `whenComplete`
lambda, defensive null check — gets a test for **each arm** in the same commit. Coverage is not
a follow-up step; letting Sonar/JaCoCo flag it later forces a second review round and produces
bolted-on coverage-only tests. The arms most often missed:

- The defensive `catch` ("should never happen, fail open") branch.
- The **success** arm of `whenComplete` (`asyncEx == null`) when only the failure path is tested.
- The `else` of an `if` guarding a rare condition.
- The null/empty input branch of a method whose happy path is the focus.

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

For members that exist only to enable testing (`resetForTest()`, exposed counters, setters on
otherwise-immutable state), the hierarchy is:

1. Don't add the affordance — restructure the test to use the real API, or move the test into
   the production class's package so default visibility suffices.
2. Package-private, with same-package tests calling it.
3. Public, only as a deliberate API decision that the hook belongs on the surface.

Promoting a test-only method to `public` because a test in another package wants it is almost
always a sign that test has fragile isolation — fix the test. A `public resetForTest()` invites
production callers and silently widens the API. If a different-package test genuinely needs the
affordance: (a) make the test same-package, (b) use a `@VisibleForTesting`-equivalent annotation
that lints other callers, or (c) accept public visibility AND document on the method why
production code must not call it.

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

## Wall-clock Timing in Tests

Do NOT gate test behaviour on wall-clock delays in the single-digit-millisecond range — JIT, GC,
and CI-runner contention routinely push two back-to-back calls past 1 ms apart, so the test
passes locally and fails on Jenkins. Control cache / throttle / TTL state explicitly instead:

- Invalidate caches in `@BeforeEach` (e.g. `cache.invalidateAll().await().indefinitely()`).
- Observe cache state directly (`keySet()`, size) rather than inferring it from timing.
- Keep TTLs at realistic production values; reset state between tests instead of shrinking the
  TTL to a number that races the JVM.

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

For waits on systems you cannot control, use Awaitility's `await()` with conservative budgets,
never raw `Thread.sleep`.

## Stubs vs. Verification

`when(...)` stubs configure behaviour — they are NOT assertions. Always use `any()` (or
`any(Type.class)`) matchers in `when(...)`; never `eq()` or raw values. Specific arguments in a
stub create the illusion of verification while asserting nothing: if the code never calls the
method, or calls it with different arguments, the stub simply doesn't match and the test still
passes. Assert arguments with `verify(...)`.

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
"everything in the universe" expression as a match argument in `when(...)` / `verify(...)`. The
matcher's meaning silently changes when the enum gains a value: the stub stops matching what
production now sends, returns `null`, and the test breaks at runtime — or worse, elsewhere. Pin
the values the test cares about, by name:

```java
// WRONG — ALL_CHANNELS = EnumSet.allOf(Channel.class) is 5 today, 6 tomorrow.
// Adding IN_APP to Channel silently changes what this matcher accepts: the stub now expects a
// 6-element EnumSet while the code under test still builds the 5 channels it actually supports —
// mock returns null, response body has nulls, assertion fails at runtime (not compile time,
// because EnumSet sizes are runtime data).
private static final Set<Channel> ALL_CHANNELS = EnumSet.allOf(Channel.class);

when(service.dispatch(any(), any(), any(), any(), anyBoolean(), any(), ALL_CHANNELS))
  .thenReturn(response);

// CORRECT — pin the EnumSet to the values the test cares about, by name.
when(service.dispatch(any(), any(), any(), any(), anyBoolean(), any(),
    EnumSet.of(Channel.EMAIL, Channel.SMS, Channel.PUSH,
      Channel.WEBHOOK, Channel.SLACK)))
  .thenReturn(response);
```

When you extend an enum / sealed type / closed set, grep the test sources
(`grep -rn "EnumSet.allOf\|.values()" src/test src/integrationTest`) and convert each wildcard
hit to an explicit enumeration. Tests that genuinely need "all current values" (e.g. a contract
test over the full API surface) are the rare exception.

## Verify with Real Argument Values

In `verify(...)`, always pass the exact expected values. `any()`-style matchers MUST NOT appear
in `verify(...)` unless the value is genuinely unknowable at test time — then add a comment
saying why.

```java
// CORRECT
verify(orderService).createOrder(orderId, customerId);

// WRONG — masks incorrect arguments
verify(orderService).createOrder(any(), any());

// Justified exception — value generated at the call site
verify(eventPublisher).publish(any(OrderCreatedEvent.class)); // UUID generated internally
```

## Asserting Absence of Interactions

Prefer `verifyNoInteractions(mock)` / `verifyNoMoreInteractions(mock)` over
`verify(mock, never()).method(...)` — `never()` guards one specific signature and passes
silently if production calls the same method with different arguments. Use `never()` only when
one specific method must not be called while other methods on the same mock legitimately are:

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

Use `ArgumentCaptor` only when the argument cannot be verified by passing the expected value to
`verify(...)` directly — typically an object constructed internally by the class under test. A
captor for a value the test can construct is noise. When a captor is genuinely needed, declare
it as an `@Captor` field, not inline via `ArgumentCaptor.forClass(...)`.

```java
// WRONG - captor is unnecessary, the value is known
@Captor ArgumentCaptor<YearMonth> monthCaptor;
verify(repository).countByMonth(monthCaptor.capture());
assertThat(monthCaptor.getValue()).isEqualTo(yearMonth);

// CORRECT - verify directly
verify(repository).countByMonth(yearMonth);
```

```java
// CORRECT - OrderRequest is built internally, cannot be reproduced from the test
@Captor ArgumentCaptor<OrderRequest> orderRequestCaptor;

verify(orderService).createOrder(orderRequestCaptor.capture());
assertThat(orderRequestCaptor.getValue().getCustomerId()).isEqualTo(customerId);
```
