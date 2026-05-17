# Agentic Coding Guidelines

This repository contains a Java 21+ microservice built with Quarkus and Gradle.

---

## MANDATORY GATES

These rules are non-negotiable. You are not permitted to skip them.

**Before writing, editing, or refactoring any Java source code:**
You MUST read the following files in full before touching any file:
1. `.agent/shared/java/rules/CODE-STYLE.md`
2. `.agent/shared/java/rules/DESIGN-PRINCIPLES.md`
3. `.agent/shared/quarkus/rules/FRAMEWORK.md`
4. If `.agent/rules/CODE-STYLE.md` exists, read it too -- project-level rules take precedence over shared ones.
5. If `.agent/rules/DESIGN-PRINCIPLES.md` exists, read it too -- project-level rules take precedence over shared ones.
6. If `.agent/rules/FRAMEWORK.md` exists, read it too -- project-level rules take precedence over shared ones.

**Before writing or modifying any test class:**
You MUST read the following files in full before touching any file:
1. `.agent/shared/java/rules/TESTING-RULES.md`
2. `.agent/shared/quarkus/rules/TESTING.md`
3. If `.agent/rules/TESTING-RULES.md` exists, read it too -- project-level rules take precedence over shared ones.
4. If `.agent/rules/TESTING.md` exists, read it too -- project-level rules take precedence over shared ones.

**Before adding, changing, or removing log statements, MDC keys, or any code that dispatches
work to another thread that needs to keep the caller's logging context:**
You MUST read `.agent/shared/java/rules/LOGGING.md` in full.
If `.agent/rules/LOGGING.md` exists, read it too -- project-level rules take precedence.

**Before running shared Gradle verification commands:**
You MUST read the following files in full before running them:
1. `.agent/shared/gradle/rules/VERIFICATION.md`
2. If `.agent/rules/VERIFICATION.md` exists, read it too -- project-level rules take precedence over shared ones.

These gates apply even for small edits. There are no exceptions.

---

## COMMIT AND PR MESSAGE GATES

**Before creating any git commit:**
You MUST read `.agent/shared/java/rules/COMMIT-AND-PR.md` in full.

**Before creating or updating a pull request title or body (`gh pr create`, `gh pr edit`):**
You MUST read `.agent/shared/java/rules/COMMIT-AND-PR.md` in full.

These gates do not apply when only writing code or running tests.

---

## MANDATORY FINAL TASKS

Every task list produced for any **code change** (writing, editing, or refactoring source code) MUST end with these
items, in this order.
These do NOT apply to non-development tasks such as creating PRs, writing documentation, or other utility tasks.
No code change is considered complete until all required items pass:

- [ ] Run all unit tests with `./gradlew test`
- [ ] Run all integration tests with `./gradlew quarkusIntTest -x test`
- [ ] Run code style and static analysis with `./gradlew check -x test -x quarkusIntTest`

### Sub-agent delegation rules

Delegate each verification task above to a sub-agent using a smaller/cheaper model when possible.
The sub-agent's job is strictly to **run, extract, and summarize**:
1. Execute the task.
2. Collect failures with full detail: file path, line number, rule/test name, error message.
3. Return a structured summary to the main agent.

The sub-agent must NOT attempt fixes. Fixing requires full codebase context and belongs
to the main agent, which will analyze the summary and decide how to proceed.

---

## 1. Task Approach

Before writing any code or running commands:
1. Check if a project-specific skill applies — look in `.agent/skills/`.
2. Clarify ambiguity - if requirements are unclear, ask instead of assuming.
3. Plan before executing - use TodoWrite for multi-step tasks.
4. Use appropriate tools - prefer specialised tools (Read, Glob, Grep) over bash for file operations.

## 2. When to Ask vs. Proceed

Ask the user when:
- Requirements are ambiguous (e.g. improve the code without direction).
- Multiple valid approaches exist and trade-offs are unclear.
- You need design or architectural decisions.
- Edge cases or error handling strategy is undefined.

Proceed when:
- The request is specific and unambiguous.
- You are following established patterns in the codebase.
- Implementing clear bugfixes or test-driven changes.
- You have sufficient context from the codebase.

## 3. File System Rules

- Always use absolute paths when performing file operations.
- Verify that files and directories exist before assuming their presence.

## 4. Common Workflow Patterns

### Bug Fix / Debugging
1. Run the relevant verification command first to understand the failure.
2. Ask clarifying questions if the bug report is vague.
3. Use TodoWrite to plan fix steps.
4. Write a test that reproduces the failure, then fix the code.
5. Verify with the MANDATORY FINAL TASKS.

### Feature Implementation
1. Clarify requirements, edge cases, and acceptance criteria with the user.
2. Use TodoWrite to plan steps before editing.
3. Add or update integration tests whenever the change affects DB code, SQL, REST clients, Kafka, OpenSearch, Redis, or similar infrastructure-facing integrations.
4. Verify with the MANDATORY FINAL TASKS.

### Code Review / Refactoring
1. Clarify the goal: what should be improved and why?
2. Use TodoWrite to plan steps.
3. Run the static-analysis check first to surface existing style or quality issues.
4. Make targeted changes, running tests after each logical change.
5. Verify with the MANDATORY FINAL TASKS.

### Documentation or Configuration Changes
1. Verify the change location and format (YAML, Markdown, properties, etc.).
2. Ask for clarification if the desired end state is unclear.
3. Make the change following existing patterns.
4. If code or build files are involved, verify with the MANDATORY FINAL TASKS.
5. Do NOT create unnecessary new files; prefer editing existing documentation.

## 5. Pull Request Workflow

### After creating a pull request

After creating a PR with `gh pr create`, monitor CI checks and code-review feedback. Address
any failures before requesting a merge.

### Addressing PR feedback

When there are review comments or CI failures to address:
1. Read the relevant rule files (see MANDATORY GATES) before editing any file.
2. Make the minimum change that addresses the feedback — do not over-fix.
3. Verify with the MANDATORY FINAL TASKS before committing.
4. Push the updated branch.
5. Reply to each comment explaining what was changed and why.
