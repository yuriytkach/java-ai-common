---
paths:
  - "**/*.java"
---

# Design Principles

Mandatory reading before writing, editing, or refactoring any Java code.

## How to read this file

Every rule below has an escape hatch — none are absolute. When a situation legitimately calls
for a deviation, take it AND record the reason in the commit message, PR description, or a
one-line comment at the deviation site. The audit trail is the rule; each rule's specific shape
is the default.

Commit and PR-message rules (subject/body format, ticket-id resolution, "files in the PR
diff are live") live in `COMMIT-AND-PR.md` — load that file when creating a commit or PR.

## Single Responsibility Principle (SRP)

Each class must have one clear reason to change. This is the most important design rule in this
codebase.

When adding a method to an existing class, ask whether it genuinely belongs to the class's
stated responsibility; if not, refactor rather than just appending. Checkstyle enforces a
1000-line hard cap — plan splits well before reaching it. When creating classes, define the
responsibility boundary up front and prefer composition over inheritance.

When a class already violates SRP and you need to add to it, do NOT silently pick an approach.
Surface the tension and ask the user to choose, then record the decision in the commit message:

- Option A: add without refactoring — violates SRP, keeps focus on the task.
- Option B: refactor first, then add — clean, but delays the feature.
- Option C: minimal targeted extraction of only the relevant behavior — the balance.

## Prefer Injectable Collaborators Over Static Helpers

When A calls B as part of A's observable behavior, B should be passed in (constructor argument,
DI bean, function value), not reached via a static method. Static calls cannot be substituted in
A's unit tests, so those tests are forced to exercise B's real logic. This applies even to pure,
stateless helpers — "stateless" does not mean "should be unmockable."

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

Genuine call-site identities — `String.format`, `Math.max`, a one-line `formatIso(...)` no test
will ever want to substitute — are fine as statics. The smell that demands injection: "I want to
unit-test A, but I have to import B's real classpath to do it."

## Follow Established Patterns Mechanically When Extending a Series

When adding the Nth entry to a series with 3+ existing entries — an alias map, error-code table,
log-key vocabulary, URL-path family, exception-name suffix, config-property prefix — first write
down the rule the existing entries follow, then apply it mechanically. Do NOT pick a name that
merely "reads nicely" next to some other identifier (e.g. a JSON field name): a pattern break
silently breaks every downstream consumer that derived its token from the pattern, even though
each file in the diff looks self-consistent. To deviate deliberately, state the reason in a
comment or the commit message.

## Match Neighbouring Conventions for New Files

The mirror rule for **file shape**: before creating a file of a kind the project already has
(`package-info.java`, `*Test.java`, `*IT.java`, a profile YAML block, a MapStruct mapper, a
Flyway migration), open two or three existing instances and copy their visible conventions —
package-level annotations, class-level suppressions, import order, test-resource setup, nesting.
Implicit conventions in neighbouring files are as binding as the explicit rules here; when they
conflict, explicit rules win. (Typical failure: three new `package-info.java` files missing the
`@ParametersAreNonnullByDefault` every neighbour declares — one minute of reading two neighbours
saves a review round.)

## Backward-Compatibility Scope

Aliases, dual code paths, fallback parsing, and deprecated-but-accepted tokens are justified only
for **live** contracts: shipped to production AND with at least one known external consumer. For
unreleased code, or internal callers you can update atomically, just change it. Before adding an
alias/fallback, name the concrete production consumer that would break without it; if you can't,
rename cleanly instead of paying permanent multi-name maintenance tax for a hypothetical caller.

## Reuse Existing Project Infrastructure Before Adding Parallel Wiring

When a feature needs a capability the project may already provide, survey what's there before
adding new wiring — a second copy of the same infrastructure drifts apart silently and every
maintainer must learn which copy to extend. Check before adding any of:

- **A declarative HTTP client** — an existing `@RestClient` (Quarkus) or `@FeignClient` /
  `WebClient` / `RestTemplate` builder (Spring) for the same upstream.
- **An `ObjectMapper` / Jackson `Module`** — there is almost always a project-wide one with the
  conventions already applied (Spring's auto-configured bean; Quarkus `ObjectMapperCustomizer`).
- **A scheduled executor** — both stacks ship `@Scheduled`; Quarkus has `ManagedScheduledExecutor`,
  Spring has `TaskScheduler`. Reuse before creating a bespoke `ScheduledExecutorService`.
- **A YAML config block** — the existing `@ConfigMapping` / `@ConfigurationProperties` interface
  may already expose the knob.

The same applies to **behavior the framework already guarantees**. Do not hand-roll a mechanism
(a `volatile` memoization field for a bean the DI container already caches, a retry loop around
a client that already retries) just because another service in the fleet does it — the other
service may predate the framework feature. Confirm the framework doesn't already provide the
behavior before adding code for it.

When new infrastructure IS the right call, prefer one shared component over a feature-local one
and document why the existing wiring did not suffice. The testing variant of this rule lives in
the framework-specific `TESTING.md`.

## Generalize Rules Instead of Enumerating Overrides

When implementing value-mapping or normalization logic, prefer a rule that generalizes —
pattern- or property-based detection — over an exhaustive hardcoded override map. A regex or
structural predicate keeps working for values it has never seen; an override map silently misses
every new value until someone notices. Keep a small explicit map only for genuine exceptions the
rule cannot express, and comment why each entry is exceptional.

## Rename Sweep — Grep, Don't Trust Pattern Replace

After renaming an identifier (public method, constant, wire token, MDC key, enum value, URL
path), run `grep -rn "<old-name>" .` and triage every hit: code and test references → rename;
documentation prose that *names* the old identifier → rename; a same-spelled but different
identifier → leave, with justification. Pattern replaces (`replace_all`, IDE find-and-replace,
`sed`) catch usage sites but routinely miss prose that describes the old name. Don't claim
"renamed" until the grep is clean or every remaining hit has a documented reason.

## Verify Before Shipping

Don't ship code whose language- or API-level semantics you haven't actually checked. "It
compiles and looks right" is not verification, especially for:

- **SQL** (PostgreSQL in particular): constructs that look right can fail at runtime —
  anonymous `DO $$…$$` blocks cannot `COMMIT`; `INSERT … RETURNING … INTO scalar` blows up on
  multi-row inserts. Read the docs for the construct or run it against a real database. Applies
  to runbooks and one-shot scripts, not only migrations.
- **Third-party APIs you haven't used, or with unclear versions**: check the actual surface of
  the version on the classpath (`javap`, the jar, pinned docs) — don't assume an API exists
  because another version had it.
- **Framework configuration parsers**: formats vary between versions and between properties
  (ISO-8601 durations vs `1ms` shorthand); a wrong format costs a CI round-trip.
- **Comparison operators across the full input domain**: prove the operator is correct on
  boundary cases. `Instant.toString()` is the textbook trap — variable sub-second precision plus
  `Z` (0x5A) sorting after `.` (0x2E) makes lexicographic order the *inverse* of chronological
  for some pairs. If the comparison can't be correct for every producible value, pin the format
  upstream (e.g. fixed-width `uuuu-MM-dd'T'HH:mm:ss.SSS'Z'`).
- **API names written into design.md / specs / tasks**: documents are contracts you implement
  against; a fictitious method name in a design doc costs a full review round. Apply the same
  `javap` / pinned-docs check to API references in docs as in code.

When you cannot verify directly, surface the assumption explicitly in the diff (comment or PR
note) so the reviewer can check it instead of CI discovering it.

## Linter Rules Are Hints, Not Directives

Static-analysis warnings are pattern matchers with no model of *why* the code exists. Before
"fixing" a warning, identify the property the existing code protects; then preserve that
property in a different shape, or document why it doesn't apply. Silently reshaping code to
appease the linter converts a style violation into a real bug.

Example: a virtual-thread body does `catch (Throwable t) { future.completeExceptionally(t); }`
so the future always settles even on `Error`. Sonar S1181 says catch `Exception`. The naive fix
breaks the guarantee — an `Error` now leaves the future hanging until timeout. The correct fix
satisfies both:

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

When a rule's heuristic genuinely doesn't apply, an explicit suppression with a justification
(`@SuppressWarnings("java:S1181") // …`, `// NOSONAR — …`) is legitimate; a silent shape-change
that drops the protection is not.

## Untestable Branches Often Signal Design Problems

If a branch is hard to drive deterministically from a test — an executor-saturation race, a
partial-write recovery that needs a stuck external call — the default response is to restructure
so the branch disappears or becomes trivially reachable, not to add a flaky concurrency test, a
hidden test seam, or a documented coverage gap. Patterns:

- **Saturated executor + `RejectedExecutionException`** → virtual threads (Java 21+): they never
  reject, and the remaining timeout path (`Future.get(timeout)`) is testable with a latch.
- **Optimistic-locking retry loop** → an atomic primitive (`compareAndSet`,
  `accumulateAndGet`) whose retry your test can drive.
- **"Should never happen" catch** → prove it can't happen and delete it, or enforce the
  precondition at the type level.
- **Defensive null-guard no caller can trigger** → delete it (see Nullability in `CODE-STYLE.md`).

The smell: you reach for `Thread.sleep`, a test-only executor setter, or `// NOSONAR coverage`.
Stop and look one level higher — can the branch be designed out? If truly not, document the
reason and a one-line manual reproduction recipe at the call site.

## Single Enforcement Boundary for Invariants

Pick one well-chosen layer to enforce an invariant and trust the other layers. Replicating the
same invariant in two or three places adds code, branches, and tests without adding safety — and
the layers drift apart. Examples:

- A `UNIQUE` DB constraint enforces uniqueness — don't also pre-check in the service.
- A SET-IF-GREATER write primitive enforces cross-writer ordering — don't also keep a per-process
  "lastFlushed" cache with a dirty-diff re-enforcing the same ordering.
- Non-null parameter types enforce non-null — see Nullability in `CODE-STYLE.md`.
- A scheduler's `concurrentExecution = SKIP` prevents overlap — don't also wrap the body in an
  `AtomicBoolean` gate.

Before adding a check, name the **specific** failure mode it defends against that the chosen
layer doesn't already cover. If you can't, delete it. "Defense in depth" is not a free pass.

## Consistency of Safety Patterns

When a class adopts a safety discipline — fail-open, idempotent retry, swallow-and-log,
defensive guard — apply it to **every** touchpoint of the same concern in the class, not only
the one you're editing. A half-applied pattern is worse than none, because callers learn to rely
on it: one cache lookup fails open while its sibling invalidate raises and breaks the request;
one JDBC call retries transient errors while its sibling against the same pool doesn't. When you
adopt or change such a pattern, sweep all related call sites in the class (and close
collaborators) and apply it uniformly; if a site genuinely shouldn't follow it, leave a one-line
comment saying why.
