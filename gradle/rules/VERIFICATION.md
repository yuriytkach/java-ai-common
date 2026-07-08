# Gradle Verification Rules

Mandatory reading before running shared Gradle verification commands.

## Verification Sequence

When verifying a code change, run these in order. A change is not complete until all pass:

1. Unit tests: `./gradlew test`
2. Integration tests: `./gradlew <integration-test-task> -x test`
3. Static analysis & style: `./gradlew check -x test -x <integration-test-task>`

`<integration-test-task>` depends on the framework:

| Framework   | Integration test task |
|-------------|-----------------------|
| Quarkus     | `quarkusIntTest`      |
| Spring Boot | `integrationTest`     |

The order matters: style checks come LAST, because fixing failing tests routinely introduces
new style violations. Re-run the style checks after every round of edits within a verification
pass — do not batch a single style check at the very end of a long editing session.

## Delegate or Run Inline

The test-runner sub-agent exists to keep long build output out of the main agent's context. It
must only **run, extract, and summarize** failures (file path, line number, rule/test name,
error message) — it must NOT attempt fixes, because fixing requires full codebase context and
belongs to the main agent. That constraint dictates when delegation pays off:

- **Delegate when you do not expect to change code based on the output**: the final full-suite
  verification, a confirmation re-run after fixes are in, reproducing a CI failure, or any long
  full-suite / integration-test run. These are pure execute-and-report steps; running them
  inline only floods the main context with minutes of build noise.
- **Run inline while actively iterating** on a write → run → fix loop. Every failure needs the
  main agent anyway, so a delegation round-trip per iteration adds latency without removing
  work.

The decision key is "will I act on the output": yes → inline, no → delegate.

## Execution Hygiene

- **Never run two `./gradlew` invocations in parallel from the same working tree.** They race
  on the shared `build/` directory and corrupt each other's outputs; the typical symptom is
  `java.nio.file.NoSuchFileException: build/classes/java/...` — it looks like a build breakage
  but is purely self-inflicted. Run them sequentially, or combine them into a single invocation
  (e.g. `./gradlew check -x <integration-test-task>`).
- **Prefer targeted static-analysis tasks when the umbrella task is blocked.** `./gradlew check`
  also compiles integration tests; an unrelated IT compilation error then masks the style
  verdict for your change. In that situation run the concrete tasks instead:
  `./gradlew checkstyleMain checkstyleTest pmdMain pmdTest spotbugsMain spotbugsTest`.
- **Testcontainers-based ITs can fail at container start with Docker network exhaustion** —
  the symptom is `all predefined address pools have been fully subnetted` or a container that
  simply won't launch after many local runs. This is environmental, not a code failure. Fix:

  ```bash
  docker network prune -f --filter "label=org.testcontainers=true"
  ```

  Re-run the suite after pruning before treating the failure as real.

## Command Execution

- Always use `./gradlew`, never a globally installed `gradle` command.
- Run Gradle commands from the repository root unless the project explicitly documents otherwise.
- Use project-level wrappers and conventions as the source of truth for task names.

## Targeted Reruns

- For unit tests, prefer `./gradlew test --tests ClassName` or `./gradlew test --tests ClassName.methodName`
  when reproducing a focused failure.
- For integration tests, use the framework-specific integration task with `--tests` when rerunning a
  specific failing IT.

## Reports

- Unit test reports are typically under `build/test-results/test` and `build/reports/tests/test`.
- Integration test reports are typically under `build/test-results/<taskName>` and `build/reports/tests/<taskName>`.
- Static analysis and style reports are typically under `build/reports`.
- When a Gradle task fails, inspect the corresponding report output and summarize the minimal actionable details:
  file path, line number, rule ID, failing test name, and error message.

## Verification Expectations

- Run the full **Verification Sequence** (above) before claiming a change is complete.
- Choose between sub-agent delegation and inline runs per **Delegate or Run Inline** (above).
  The main agent always remains responsible for code changes.

## Schema / Enum / DTO Changes Require Full-Suite Verification

When you change anything that ripples through the type system — adding a field to a record,
adding a value to an `enum`, adding a parameter to a constructor or a public method, narrowing or
widening a generic bound — scoped test runs (`./gradlew test --tests "*MyClass*"`) are NOT enough.
Run the FULL unit suite AND the full integration suite before claiming the change is verified.

The mistake shape: you add a 6th field to a record. The compiler catches every constructor-call
site (~29 of them), you fix the arity, and you run `./gradlew test --tests "*ChangedClassTest*"`
— it passes. Meanwhile, an unrelated test uses `EnumSet.allOf(Channel.class)` as a Mockito stub
matcher; the matcher's runtime cardinality silently changed under your enum addition; the test
fails at runtime but you don't see it because you only ran scoped tests. CI catches it. Round-trip
wasted.

For changes of this kind, the mandatory verification before commit is:

- `./gradlew test` — the full unit suite, unscoped.
- `./gradlew <integration-test-task>` — the full IT suite, unscoped. (Task name varies per
  framework — see the **Verification Sequence** table above.)
- `./gradlew check -x test -x <integration-test-task>` — static analysis in isolation.

Scoped reruns are for **reproducing** a specific failure, not for **verifying** a schema-level
change.

Triggers that demand full-suite verification:

- Adding / removing / renaming a record component.
- Adding / removing an `enum` value.
- Adding / removing a constructor parameter or method parameter on any non-private API.
- Renaming a public method, field, or class.
- Changing a public method's return type.
- Touching a class that is referenced via `@InjectMocks` or as a Mockito stub target in 5+ tests.

## Linter Budgets Are Tech-Debt Ceilings, Not Headroom

Many Gradle projects configure linter tasks with explicit error budgets:

```groovy
checkstyleMain.maxErrors = 2
checkstyleTest.maxErrors = 6
checkstyleIntegrationTest.maxErrors = 1
pmdTest.maxFailures = 0
```

These budgets exist because the project has pre-existing violations the team has chosen to live
with (often: one or two `FileLength` violations on legacy classes, or `TooManyMethods` on a test
class that's already large). **Your additions must NOT push the violation count up.** A budget of
`maxErrors = 1` means "we tolerate the one violation that's already there"; it does NOT mean "we
have headroom for one more."

The mistake shape: a test class is already 987 lines (under the 1000-line `FileLength` cap). You
add 95 lines of new tests, taking the file to 1082 lines. Checkstyle now reports 2 errors against
a `maxErrors = 1` budget. CI fails. You shrug, bump the budget to 2 in `build.gradle`, and push.
Wrong: the budget is the project's tolerance for pre-existing debt, not a knob to widen so your
new code can fit.

Before committing, check whether your additions cross any of the structural limits:

- **File length** (Checkstyle `FileLengthCheck`, typically `max = 1000`). Run
  `wc -l <changed-files>` and compare against the limit.
- **Method count per class** (PMD `TooManyMethods`, typically threshold 10). Especially for test
  classes with many `@Test` methods; nested `@Nested` test classes are NOT a workaround in DBRider
  contexts (see project testing rules).
- **Line length** (Checkstyle `LineLength`, typically 120 chars). Auto-formatters won't always
  catch long `@ValueSource` / annotation lines.
- **Cyclomatic complexity / `NPath` / `CognitiveComplexity`** thresholds when adding branching to
  an already-busy method.

If a structural limit is in your way, **split** the file / class / method, don't increase the
budget. Reserve budget bumps for cases where the entire team has agreed the limit itself is wrong.

Run static checks BEFORE committing, with the same tasks CI uses:

```
./gradlew check -x test -x <integration-test-task>
```

…and confirm `BUILD SUCCESSFUL`. If a violation appears, the fix lives in your branch — not in
`build.gradle`'s `maxErrors` budget.
