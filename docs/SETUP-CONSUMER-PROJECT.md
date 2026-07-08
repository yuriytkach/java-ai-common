# Consumer Project Setup For Coding Agents

Use this document when a coding agent needs to connect `java-ai-common` to a consumer repository.

## Goal

Set up this repository as a git submodule at `.agent/shared`, configure sparse checkout for the
consumer project's framework, create the root `AGENTS.md` and `CLAUDE.md` symlinks and the
`.claude/rules/shared` symlink for Claude Code path-scoped rules, and update the consumer
project's `README.md` with the shared AI agent setup instructions.

## Required input

The user must provide the git URL for this repository.

Example:

```text
git@github.com:<your-org>/java-ai-common.git
```

## What the agent must do

1. Detect whether the consumer project is Quarkus or Spring Boot.
2. Add the git submodule at `.agent/shared` if it does not already exist, or initialise the existing
   submodule checkout if it is already present.
3. Configure or repair sparse checkout for the required shared directories.
4. Create or update the root `AGENTS.md` and `CLAUDE.md` symlinks.
5. Create or update the `.claude/rules/shared` symlink so Claude Code auto-loads the path-scoped
   rules.
6. Add the AI agent setup section to the consumer project's `README.md`.
7. Verify the resulting setup and summarize what changed.

## Framework detection

Detect the project type from its build files and code layout.

- Choose `quarkus` when the project clearly uses Quarkus.
- Choose `spring-boot` when the project clearly uses Spring Boot.
- Ask the user only if the project type is ambiguous.

## Sparse checkout paths

- Quarkus project: `java gradle quarkus`
- Spring Boot project: `java gradle spring-boot`

## Commands

Add the submodule if needed:

```bash
git submodule add <REPO_URL> .agent/shared
```

Initialise the submodule if it already exists but is not checked out:

```bash
git submodule update --init --recursive
```

If the submodule already exists and the consumer project should adopt the latest shared guidance,
advance it deliberately and then commit the updated submodule pointer in the consumer repository:

```bash
git submodule update --remote --recursive
git add .agent/shared
git commit -m "Bump agent-shared to latest"
```

Inspect the current sparse checkout before deciding whether it needs repair:

```bash
git -C .agent/shared sparse-checkout list
```

Configure sparse checkout for a Quarkus project:

```bash
git -C .agent/shared sparse-checkout init --cone
git -C .agent/shared sparse-checkout set java gradle quarkus
```

Configure sparse checkout for a Spring Boot project:

```bash
git -C .agent/shared sparse-checkout init --cone
git -C .agent/shared sparse-checkout set java gradle spring-boot
```

Create the root `AGENTS.md` symlink for a Quarkus project:

```bash
ln -s .agent/shared/quarkus/AGENTS.md AGENTS.md
```

Create the root `CLAUDE.md` symlink for a Quarkus project:

```bash
ln -s .agent/shared/quarkus/AGENTS.md CLAUDE.md
```

Create the root `AGENTS.md` symlink for a Spring Boot project:

```bash
ln -s .agent/shared/spring-boot/AGENTS.md AGENTS.md
```

Create the root `CLAUDE.md` symlink for a Spring Boot project:

```bash
ln -s .agent/shared/spring-boot/AGENTS.md CLAUDE.md
```

Create the `.claude/rules/shared` symlink so Claude Code auto-loads the path-scoped rules
(`claude-rules/` carries the rule files' `paths:` frontmatter). For a Quarkus project:

```bash
mkdir -p .claude/rules
ln -s ../../.agent/shared/quarkus/claude-rules .claude/rules/shared
```

For a Spring Boot project:

```bash
mkdir -p .claude/rules
ln -s ../../.agent/shared/spring-boot/claude-rules .claude/rules/shared
```

## Existing setup remediation

If `.agent/shared` already exists, the agent must verify and repair the existing setup instead of
assuming it is correct.

1. Ensure the submodule working tree is present:

   ```bash
   git submodule update --init --recursive
   ```

2. Inspect the current sparse checkout:

   ```bash
   git -C .agent/shared sparse-checkout list
   ```

3. Reapply the framework-specific sparse checkout so the required directories are definitely present:

   ```bash
   git -C .agent/shared sparse-checkout init --cone
   git -C .agent/shared sparse-checkout set java gradle <FRAMEWORK>
   ```

4. Verify the required framework file exists inside the submodule:

   ```bash
   test -f .agent/shared/<FRAMEWORK>/AGENTS.md
   ```

5. Verify the root symlinks:

   ```bash
   test "$(readlink AGENTS.md)" = ".agent/shared/<FRAMEWORK>/AGENTS.md"
   test "$(readlink CLAUDE.md)" = ".agent/shared/<FRAMEWORK>/AGENTS.md"
   ```

6. Verify (or create) the `.claude/rules/shared` symlink:

   ```bash
   test "$(readlink .claude/rules/shared)" = "../../.agent/shared/<FRAMEWORK>/claude-rules"
   ```

7. If the required framework file is missing because the pinned submodule commit is too old, or if
   the user explicitly wants the latest shared guidance, advance the submodule with:

   ```bash
   git submodule update --remote --recursive
   git add .agent/shared
   git commit -m "Bump agent-shared to latest"
   ```

## `AGENTS.md` and `CLAUDE.md` handling rules

- `AGENTS.md` and `CLAUDE.md` should both point to `.agent/shared/<framework>/AGENTS.md`.
- If either file does not exist, create the framework-specific symlink.
- If either file already exists as the correct symlink, leave it as is.
- If either file exists as a symlink to the wrong framework, replace it with the correct symlink.
- If either file exists as a regular file, do NOT overwrite it automatically. Ask the user whether to keep
  the project-specific file or replace it with a symlink.
- `.claude/rules/shared` should point to `.agent/shared/<framework>/claude-rules`. Create it if
  missing; if it points to the wrong framework, replace it. This is what lets Claude Code
  auto-load the path-scoped rules.

## `README.md` update requirements

Add or update a section named `AI Agent Setup` in the consumer project's `README.md`.

That section should include:

- the fact that this repository is connected as a git submodule at `.agent/shared`
- the sparse checkout paths required for the project's framework
- the root `AGENTS.md` and `CLAUDE.md` symlink targets
- the `.claude/rules/shared` symlink target for Claude Code path-scoped rules
- instructions for `git pull --rebase --recurse-submodules`
- instructions for `git submodule update --remote --recursive`
- the recommended git aliases `pullr` and `sub-update`

If the `README.md` already has an AI-agent setup section, update it instead of duplicating it.

## Verification

After making the changes, verify:

1. `.agent/shared` exists and is a git submodule
2. sparse checkout is configured for the expected paths
3. `AGENTS.md` and `CLAUDE.md` point to the correct framework file
4. `.claude/rules/shared` points to the correct framework's `claude-rules` folder
5. `README.md` contains the AI agent setup section

## Expected summary

The agent should report:

- detected project type
- whether the submodule was added or reused
- which sparse checkout paths were configured
- which `AGENTS.md` and `CLAUDE.md` targets were created or kept
- whether the `.claude/rules/shared` symlink was created or kept
- whether `README.md` was created or updated
- any manual follow-up needed from the user
