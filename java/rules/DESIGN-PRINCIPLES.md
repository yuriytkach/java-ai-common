# Design Principles

Mandatory reading before writing, editing, or refactoring any Java code.

## How to read this file

Every rule below has an escape hatch — none of them are absolute. When the situation legitimately
calls for a deviation, take it AND record the reason in the commit message, PR description, or a
one-line code comment at the deviation site. The audit trail is the rule; the specific shape of
each rule is the default.

Commit and PR-message rules (subject/body format, ticket-id resolution, "files in the PR
diff are live") live in `COMMIT-AND-PR.md` — load that file when creating a commit or PR.

## Single Responsibility Principle (SRP)

Each class must have one clear reason to change. This is the most important design rule in this codebase.

### When Adding Methods to Existing Classes

1. Review the existing class structure and its stated responsibility.
2. Ask: does this new method genuinely belong to that responsibility?
3. If not, do not just add it - consider refactoring first (see below).
4. After implementation, verify the class still has a single, clear purpose.
5. Hard limit: Checkstyle enforces a 1000-line maximum. Plan refactoring well before reaching it.

### Handling Legacy Code and Refactoring Trade-offs

When a class already violates SRP and you need to add to it, surface the tension explicitly.
Present the user with options before proceeding:

- Option A: Add the method without refactoring - violates SRP but keeps focus on the current task.
- Option B: Refactor first, then add - ensures SRP but delays the feature.
- Option C: Minimal targeted refactoring that does not disrupt the wider codebase - a balance.

Do not silently pick an option. Ask the user to decide, then document the decision in the commit message.

Example prompts:
- Small case: This class has 2 responsibilities. Should I extract the new logic into a separate class first, or just add it?
- Large case: This class has 5 responsibilities and is 800 lines. Full refactoring would affect 10+ classes.
  Should I: (A) just add the method, (B) do a full refactor first, or (C) extract only the relevant behavior into a new class?

### When Creating New Classes

- Define clear responsibility boundaries from the start.
- If a class grows large, consider splitting it proactively before hitting limits.
- Prefer composition over inheritance for combining behaviour.

## Prefer Injectable Collaborators Over Static Helpers

When code A calls code B as part of A's observable behavior, B should be **passed in** (constructor
argument, DI bean, function value) rather than reached via a static method on a utility class.
Static-method calls cannot be substituted in unit tests of A; tests of A then necessarily exercise
B's real logic, which inflates the test surface and couples A's failure isolation to B's.

This applies even to pure, stateless helpers. "Stateless" does not mean "should be unmockable." If a
test of the caller wants to assert "A invoked B with X," B needs to be a substitutable seam.

```java
// WRONG — static utility, callers' tests can't substitute it
final class FlushPlanner {
  private FlushPlanner() { }
  static List<PublishGroup> plan(final FlushSnapshot snap) { ... }
}

class Recorder {
  void flush() {
    final var groups = FlushPlanner.plan(takeSnapshot());
    ...
  }
}

// CORRECT — collaborator is injected, callers' tests can substitute it
@ApplicationScoped               // or @Service / @Component depending on the stack
class FlushPlanner {
  List<PublishGroup> plan(final FlushSnapshot snap) { ... }
}

@ApplicationScoped               // or @Service / @Component depending on the stack
@RequiredArgsConstructor
class Recorder {
  private final FlushPlanner planner;
  void flush() {
    final var groups = planner.plan(takeSnapshot());
    ...
  }
}
```

Genuinely call-site identities — `String.format`, `Math.max`, a one-line `formatIso(...)` whose
behavior tests are never going to want to substitute — are fine as static methods. Prefer
injection **when test substitutability matters** (i.e. when callers' tests will want to assert
"A invoked B with X"). The recurring smell is "I want to write a unit test for A, but I have to
import B's real classpath to do it."

## Follow Established Patterns Mechanically When Extending a Series

When adding the Nth entry to a series that already has 3+ entries — an alias map, an error-code
table, a log-key vocabulary, a URL-path family, an exception-name suffix convention, a config-
property prefix — derive the new value from the existing rule mechanically. Do NOT pick a name
that "reads nicely" alongside some other identifier (e.g. matching the JSON field name) until you
have first computed what the existing pattern would dictate and explicitly chosen to deviate.

The mistake shape: 5 entries follow Pattern A (`USERS → users`, `UNIQUE_CLIPPERS → uniqueClippers`,
i.e. enum-name → lowerCamelCase). When you add the 6th, you reach for a name that pairs nicely
with the JSON field name (`anyActivityUserCount`) instead of the Pattern-A value
(`anyActivityUsers`). The pattern break breaks any downstream consumer that derived its token
from Pattern A, even though every individual file in the diff looks self-consistent.

Before adding a new entry to a series:

1. List the existing entries and write down the rule that maps them.
2. Mechanically apply the rule to the new value.
3. If you want to deviate, state the reason in a comment or commit message.

Series this applies to (non-exhaustive): `Map<String, Enum>` alias tables, `Enum.name() →` wire
tokens, error-code enums, log MDC key names, URL path segments, configuration property keys,
exception class name suffixes, REST resource sub-paths.

## Match Neighbouring Conventions for New Files

The "Follow Established Patterns Mechanically" rule above governs the *content* of a sequence
of similar values (enum aliases, error codes, log keys). The mirror rule applies to **file
shape**: when adding a file of a kind that already exists in the project — `package-info.java`,
`*Test.java`, `*IT.java`, a profile-specific YAML block (Quarkus `%test` / Spring
`application-test.yml`), a Gradle module, a Mapstruct mapper, a Flyway migration — open at least
two existing instances and copy whatever conventions are visible in them: package-level
annotations, class-level `@SuppressWarnings`, the order of imports, the test-resource choice
(`@QuarkusTestResource` for Quarkus, `@Testcontainers` / `@TestConfiguration` for Spring), the
inner-class organization, the indentation style. **The implicit project conventions visible in
neighbouring files are as binding as the explicit ones in the rule docs, unless an explicit rule
in this doc contradicts the local convention — explicit rules win.**

The mistake shape: every existing `package-info.java` in `src/main/java/.../com/example/service/`
declares `@ParametersAreNonnullByDefault`. You add three new ones without it because the rule
docs do not mandate it. A reviewer flags all three; one minute of reading two neighbours at the
start would have saved a review round.

Before creating a new file:

1. `find <module>/src -name '<same-shape>.<ext>' | head -3` and open two or three of them.
2. Note the per-file boilerplate (package annotations, imports, class-level suppressions,
   nested structure).
3. Mirror it.

## Backward-Compatibility Scope

Backward-compatibility affordances — aliases, dual code paths, fallback parsing, deprecated-but-
still-accepted tokens — are only justified for **live** contracts: code that has already shipped
to production AND has at least one known external consumer. For code that hasn't been released,
or that only has internal callers you also control and can update atomically, just change it.

The mistake shape: a reviewer points out that the wire token in a new (unreleased) PR is named
wrong. You "fix" it by accepting both the wrong and the right token "in case anyone has already
started integrating." Nobody has — the code doesn't exist yet — and now you have permanent
multi-name baggage that future maintainers have to wire through every alias map, error message,
log token, and test.

Before adding an alias / fallback / dual-name affordance, answer:

- Is there a concrete production consumer that would break without it? (Name it.)
- If not, just rename. Don't pre-emptively pay the maintenance tax for a hypothetical caller.

## Reuse Existing Project Infrastructure Before Adding Parallel Wiring

When the feature you are about to ship needs a capability the project may already provide —
an HTTP client builder, a configuration mapping, a JSON `ObjectMapper`, a scheduled executor,
a metrics registry — your first move is to **survey what's already there**, not to add new
wiring. Adding a second copy of the same infrastructure costs more than the file you wrote:
every future maintainer now has to learn which copy to extend, and the two copies drift apart
silently.

Before adding any of the following, grep / read the existing project state to confirm it
isn't already provided:

- **A new declarative HTTP client.**
  - Quarkus: check for an existing `@RestClient` interface bound to the same upstream.
  - Spring: check for an existing `@FeignClient`, `WebClient` bean, or `RestTemplate` builder
    targeting the same upstream.
- **A new `ObjectMapper` / Jackson `Module`** — there is almost always a project-wide one with
  the project's conventions (date format, naming strategy, enum handling) already applied.
  Spring exposes it as the auto-configured `ObjectMapper` bean; Quarkus exposes it via
  `quarkus-jackson` / `ObjectMapperCustomizer`.
- **A new scheduled executor.** Both stacks ship `@Scheduled`; Quarkus also exposes a
  `ManagedScheduledExecutor`, Spring exposes a `TaskScheduler` bean. Reuse before creating a
  bespoke `ScheduledExecutorService`.
- **A new YAML config block.** Check whether the existing `@ConfigMapping` (Quarkus) or
  `@ConfigurationProperties` (Spring) interface for the feature already exposes a knob you can
  reuse.

When introducing infrastructure is the right call, prefer adding a single shared component over
a feature-local one, and document why the existing wiring did not suffice.

The same principle applies to test infrastructure (test resources, test containers, base IT
classes) — see the framework-specific `TESTING.md` for the testing variant of this rule.

## Rename Sweep — Grep, Don't Trust Pattern Replace

After renaming an identifier (a public method, a constant, a wire token, an MDC key, an enum value,
a URL path), run a full-text search of the entire repo for the OLD name and classify every hit:

- Code reference → rename.
- Test / fixture reference → rename.
- Documentation prose that *describes* the old name ("the canonical token is X", "URL token X
  maps to log token Y") → rename.
- A reference to a different identifier that happens to spell the same thing → leave, but justify.

Pattern-based replacements (the `Edit` tool with `replace_all: true`, IDE find-and-replace,
`sed -i 's/oldname/newname/g'`) catch the syntactic shape of usage sites but routinely miss
documentation that *names* the old identifier in descriptive prose. After every rename, the
verification ritual is:

```
grep -rn "<old-name>" .
```

…and a manual triage of every result. Don't claim "renamed" until that grep is clean or every
remaining hit has a documented reason.

## Verify Before Shipping

Don't ship code whose language- or API-level semantics you haven't actually checked. "It compiles
and looks right" is not a substitute for verification, especially for:

- **SQL** (PostgreSQL in particular). Some constructs that look correct will fail at runtime —
  e.g. anonymous `DO $$ … $$` blocks cannot issue `COMMIT` (only `CREATE PROCEDURE` bodies can), and
  `INSERT … RETURNING … INTO scalar_var` blows up when the INSERT produces more than one row.
  Read the docs for the construct, or run it against a real database, before treating SQL as done.
  This applies to operator runbooks and one-shot scripts, not only to Flyway migrations.
- **Third-party APIs you haven't used before, or whose version is unclear.** Check the actual public
  surface of the runtime version you are targeting — don't assume an API exists just because it was
  available in another version of the same library. Inspect the jar (`javap`, IDE), the library's
  release notes, or the documentation for the exact version on the classpath.
- **Framework configuration parsers.** Configuration formats vary subtly between framework versions
  and even between properties inside the same framework (e.g. ISO-8601 duration strings vs. shorthand
  like `1ms`). A wrong format usually fails at build/boot rather than at runtime, but it still costs
  a CI round-trip if not verified locally.
- **Comparison operators across the full input domain.** When the correctness of a comparison
  (`>`, `<`, `compareTo`, lexicographic / numeric / Unicode-collation, Lua `>` on strings, SQL
  `ORDER BY` against text) depends on the relationship between two values, prove the operator
  does the right thing on the **boundary cases** — empty, longest, shortest, edge of representable
  values, format variations. `Instant.toString()` is the textbook example: it produces variable
  sub-second precision and the `Z` character (0x5A) sorts AFTER `.` (0x2E), so
  `"2026-05-15T10:00:00Z"` sorts AFTER `"2026-05-15T10:00:00.000000001Z"` lexicographically —
  the exact inverse of chronological order. If you cannot make the comparison correct for every
  value the domain can produce, pin the format upstream (e.g. fixed-width `uuuu-MM-dd'T'HH:mm:ss.SSS'Z'`)
  until you can. Same principle applies to user-supplied locale-dependent text, mixed unicode
  normalization forms, and numeric strings with leading zeros.
- **API names you write into design.md, spec.md, or task descriptions.** Documents are part of
  the contract you implement against, and a fictitious method name in a design doc costs a full
  PR review round to discover and unwind. Apply the same `javap` / IDE / pinned-docs check to
  API references in docs as you would in code. For example, if the bytecode shows a client class
  exposing `execute(String, String...)` but no `script()` method, do not write
  `client.script().scriptLoad(...)` into design.md — invent neither methods nor signatures.

When you cannot run or directly verify the code yourself, surface the assumption explicitly in the
diff (a comment, a PR description note) so the reviewer can verify it on your behalf rather than
discovering the mistake in CI.

## Linter Rules Are Hints, Not Directives

Static-analysis warnings (Sonar, PMD, SpotBugs, Checkstyle, ESLint, etc.) are pattern matchers with
no model of *why* the code under review exists. Before "fixing" a warning, identify the property
the existing code was protecting. Then either preserve that property with a different shape, or
document explicitly why the property does not apply here. Silently changing the code's shape to
make the linter happy — without re-establishing the underlying property — converts a style
violation into a real bug.

The mistake shape: code inside a virtual-thread body does `catch (Throwable t) { future.completeExceptionally(t); }`
so the surrounding `CompletableFuture` always settles, even if `task.run()` throws an `Error`
(`StackOverflowError`, `AssertionError`, OOM). Sonar S1181 flags "Catch Exception instead of
Throwable." The naive fix — `catch (Exception ex)` — silences the warning AND breaks the
load-bearing property: an `Error` now bypasses the catch, the future is never completed, and the
caller's `Future.get(timeout)` blocks for the full timeout for no benefit. The correct fix
satisfies BOTH the rule and the property:

```java
// CORRECT — catches Exception (S1181 satisfied) and still guarantees completion on Error
boolean settled = false;
try {
  task.run();
  future.complete(null);
  settled = true;
} catch (final Exception ex) {
  future.completeExceptionally(ex);
  settled = true;
} finally {
  if (!settled) {
    future.completeExceptionally(
      new IllegalStateException("task ended without normal completion"));
  }
}
```

When a rule's heuristic genuinely does not apply, an explicit suppression with a justification
(`@SuppressWarnings("java:S1181") // catching Throwable here to ...`, `// NOSONAR — ...`) is a
legitimate response. A silent shape-change that drops the protection is not.

## Untestable Branches Often Signal Design Problems

If a branch in the code is hard to drive deterministically from a unit test — `RejectedExecutionException`
from an executor saturation race, a `NOSUCHELEMENT` race between two threads, a partial-write
recovery path that requires a stuck Redis call to reproduce — that is often a signal about the
**design**, not about the test. The default response is to restructure the code so the branch
either disappears or becomes trivially reachable, not to add a flaky concurrency test, a hidden
test seam, or a documented coverage gap.

Concrete restructure patterns that eliminate untestable branches:

- **Saturated executor + `RejectedExecutionException`** → switch to virtual threads
  (Java 21+) or to a primitive whose contract makes rejection impossible. Java 21
  `Thread.ofVirtual().start(...)` never rejects; the only timeout enforcement remaining is
  `Future.get(timeout)`, which is deterministically testable with a `CountDownLatch`.
- **Optimistic-locking retry loop** → an atomic primitive (`AtomicReference.accumulateAndGet`,
  `compareAndSet` with a single retry expressed as a loop your test can stub) instead of a
  busy-retry over an external resource.
- **"Should never happen" catch block** → either prove statically it cannot happen and delete
  the catch, or restructure the caller so the precondition is enforced at the type level.
- **Defensive null-guard inside a method whose callers cannot pass null** → delete the guard.
  See the Nullability rule in `CODE-STYLE.md`.

The recurring smell is "I want to assert this branch behaves, but I can't drive the input that
selects it without weakening either the production code or another test." When you reach for
`Thread.sleep`, a custom test-only executor injected via setter, or a `// NOSONAR coverage`
annotation, stop and look one level higher: can the branch itself be designed out of existence?
If it cannot, document the reason and a one-line manual reproduction recipe at the call site.

## Single Enforcement Boundary for Invariants

Pick one well-chosen layer to enforce an invariant and trust the other layers. Replicating the
same invariant in two or three places does not make the system safer; it adds code, branches,
tests, and a continuous risk of divergence between the layers.

Examples of single-boundary placement:

- A `UNIQUE` DB constraint enforces uniqueness. Don't also pre-check uniqueness in the service.
- A SET-IF-GREATER write primitive on a shared store enforces monotonic ordering across writers.
  Don't also keep a per-process "lastFlushed" cache + dirty-diff that re-enforces ordering.
- The type system enforces non-null for parameters typed non-null — see the Nullability rule in
  `CODE-STYLE.md`.
- A scheduler's `concurrentExecution = SKIP` prevents overlapping ticks. Don't also wrap the body
  in a per-process `AtomicBoolean` "is running" gate.

The mistake shape: a multi-pod service publishes to a Redis hash through a SET-IF-GREATER Lua
script that already enforces "newer write wins" cross-pod. The implementer adds in-memory
`AtomicReference<Instant>` with a CAS-max retry loop on every `record*` method, *plus* a
`lastFlushed` mirror per stat, *plus* a dirty-diff before each flush — three independently-correct
mechanisms enforcing the same invariant the Lua script already enforces. Reviewers, tests, and
operators now have to keep all four in sync.

Before adding a check, name the **specific** failure mode it defends against that the chosen
enforcement layer doesn't already cover. If you can't, delete it. "Defense in depth" is not a
free pass; each defense costs maintenance, and they tend to drift apart as the system evolves.

## Consistency of Safety Patterns

When a class adopts a safety discipline — fail-open, idempotent retry, swallow-and-log, defensive
null guard — apply it to **every** touchpoint of the same concern in the class, not only the one
you happen to be editing. A half-applied pattern is worse than no pattern because callers learn to
rely on it.

Common shapes of this mistake:

- One cache lookup is wrapped in a fail-open `try/catch`; the sibling cache invalidate / put on the
  failure path is not. The first publish "fails open"; the cleanup raises and breaks the request.
- One JDBC call retries on transient connection errors; a sibling JDBC call in the same service
  does not, even though both face the same connection pool.
- One DTO mapper null-guards an optional field; a sibling mapper for the same domain crashes on
  the same input.

When you adopt or change a safety pattern in a class, sweep all related call sites in the same
class (and any close collaborator) and apply the pattern uniformly. If a site genuinely should NOT
follow the pattern, leave a one-line comment explaining why.
