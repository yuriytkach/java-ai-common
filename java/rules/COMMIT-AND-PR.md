# Commit and Pull Request Messages

Mandatory reading before creating any git commit, pull request title, or pull
request body.

## Commit subject

Format: `<type>: <Capitalized imperative subject>`

- Length: aim for ≤50 chars; never exceed 72. The GitHub-injected `(#1234)`
  suffix on squash merge does not count toward the limit.
- Imperative mood (`Add`, `Fix`, `Remove` — not `Added`/`Adds`/`Adding`).
- Capitalize the first letter after the colon.
- No trailing period.
- One blank line between subject and body.

Do NOT add a scope in parentheses (e.g. `feat(audience-activity): ...`).
The team uses bare types only. Scopes that appear in some recent history are
artefacts of unedited tooling output, not the convention.

### Type prefix (required on `main`/`master`; recommended elsewhere)

| Type       | Use for                                                          |
|------------|------------------------------------------------------------------|
| `feat`     | New functionality not present before                             |
| `fix`      | Bug fix                                                          |
| `doc`      | Documentation only                                               |
| `refactor` | Code restructure or style change with no behavior change         |
| `perf`     | Performance improvement                                          |
| `test`     | Adding or correcting tests, test utilities, test fixtures        |
| `build`    | Build system, dependencies, CI/CD (Jenkins, Terraform, Gradle)   |
| `revert`   | Reverts a prior commit                                           |
| `chore`    | Anything that does not fit the above                             |

## Commit body

- Wrap at 72 columns.
- Explain **what** changed at a high level and, more importantly, **why** —
  the rationale, motivation, or problem being solved. Do not explain *how*;
  the diff already shows that.
- Skip the body only when the subject is fully self-explanatory
  (e.g. `chore: Remove unused runbook`).
- Do not pad with `This commit…` / `This change…`.
- Do not include a test plan, file list, or change checklist.

### Jira reference

Add the Jira ticket id on its own line as the last line of the body.

Resolve the ticket id in this order:

1. **Branch name.** If `git branch --show-current` matches
   `<initials>.<TICKET-ID>.<topic>` (e.g. `jd.proj-1234.refactor-cache`),
   extract the middle segment and uppercase it: `PROJ-1234`. This is the
   common case — derive it automatically; do not ask the user.
2. **User-provided hint.** If the commit invocation (e.g. `/commit PROJ-1234 …`)
   contains a ticket id, that wins over the branch-derived one.
3. **Fallback.** If no id can be resolved, use the literal `NOJIRA`. Do not
   guess from the diff or recent commits.

Examples:

```
feat: Add anyActivityUserCount metric

Add a sixth metric to the admin endpoint, sourced from user_last_activity.
The monthly report needs both the legacy count and the new any-activity
count as separate rows.

PROJ-7373
```

```
refactor: Remove duplication in controller files

NOJIRA
```

## Pull request title

Same format as the commit subject (`<type>: <Capitalized imperative subject>`).

- Do NOT put the Jira ticket id in the title.
- Do NOT add a scope in parentheses — same rule as commits.
- Keep it short. The title is a label, not a summary.
- If the PR contains one logical commit, the title may match that commit's
  subject verbatim.

## Pull request body

A PR may bundle several commits. The body explains the change at PR level,
not commit level. Be concise; reviewers read the diff for details.

### Required structure

1. **Why** — the rationale. Lead with this. One short paragraph or 2-4 bullets:
   the problem, the trigger, the constraint that forced the change. Without
   this section reviewers cannot judge whether the approach is correct.
2. **What** — a brief summary of the change. Bullets are fine. Stay
   high-level; do not narrate every file.
3. **Notes** (optional) — anything reviewers should specifically look at:
   risks, gotchas, follow-up items, out-of-scope decisions, manual ops steps,
   and links to related PRs in other repositories (stacked PRs, cross-service
   contract changes).

Do NOT include a "Test plan" section. CI verification is owned by the
pipeline, not the PR body.

### Length and line wrapping

- **Scale to the change.** A two-file refactor gets a two-line Why and a
  one-bullet What. A reviewer should be able to read the whole body in
  under 30 seconds for most PRs. Do not pad to hit a perceived "PR body
  length"; do not repeat the Why in different words inside the What.
- **Do NOT hard-wrap PR body lines.** Unlike commit bodies (which are read
  in terminals and `git log`), PR descriptions are rendered in a browser.
  Write each paragraph as one continuous line and let the renderer wrap.
  Use blank lines to separate paragraphs and bullets.

### Example

```markdown
## Why

The active-coupons cache sometimes returns an empty list for users who do have active coupons. Root cause is a non-atomic `SMEMBERS + EXISTS` pair — a concurrent write from another pod can land between the two Redis calls.

## What

- Treat empty `SMEMBERS` as a cache miss; drop the second round-trip.
- Add an integration test exercising 50 trials × 4 readers + 1 writer against real Redis.

## Notes

- Race has been latent since PR #1148.
- No DB or API contract changes.
```

## Files in the PR Diff Are Live, Not Historical

Any file that appears in the current PR's diff — including files under
`docs/decisions/`, `CHANGELOG.md`, sample data,
or any other "historical" directory — is part of the PR's reviewed contract.
It will ship to main with this PR. If a contract decision changes during PR
review, the change must propagate to those files too.

"It's in `archive/`" is not a license to leave stale claims in the PR. Either:

- Update the historical file's content to match the final contract, OR
- Add a small "post-archive amendment" / "supersedes" pointer that directs
  readers to the authoritative source.

A reviewer who finds the archive contradicting the canonical spec inside the
same PR is correct; the archive is part of what's being reviewed, even if it
conceptually represents an earlier moment.
