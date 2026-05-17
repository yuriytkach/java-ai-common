# java-ai-common

Shared AI coding guidance for Java projects.

This repository is intended to be mounted into consumer repositories under `.agent/shared`.

## Layout

- `java/` — shared Java code-style, design, and testing rules
- `gradle/` — shared Gradle verification guidance
- `quarkus/` — Quarkus-specific `AGENTS.md` and framework/testing rules
- `spring-boot/` — Spring Boot-specific `AGENTS.md` and framework/testing rules

Consumer repositories are expected to symlink both their root `AGENTS.md` and `CLAUDE.md` to the
appropriate framework file, for example `.agent/shared/quarkus/AGENTS.md` or
`.agent/shared/spring-boot/AGENTS.md`.

## AI Agent Setup

Consumer projects add this repository as a git submodule at `.agent/shared`. The submodule can
use sparse checkout so each project pulls only the shared directories it needs.

For an agent-assisted setup, use the guide in `docs/SETUP-CONSUMER-PROJECT.md`.

### Required sparse checkout paths

- Quarkus projects: `java`, `gradle`, `quarkus`
- Spring Boot projects: `java`, `gradle`, `spring-boot`

### Consumer project layout example

```text
my-service/
  .agent/
    shared/                # git submodule pointing to this repository
      java/
      gradle/
      quarkus/             # or spring-boot/
    rules/                 # optional project-specific rule overrides
  AGENTS.md                # symlink to .agent/shared/<framework>/AGENTS.md
  CLAUDE.md                # symlink to the same .agent/shared/<framework>/AGENTS.md
```

Notes:

- `.agent/shared/` contains the shared guidance from this repository.
- `.agent/rules/` is optional and intended for project-specific rule overrides (files with the
  same name as shared rule files take precedence over the shared ones).
- `AGENTS.md` and `CLAUDE.md` in the project root are usually symlinks to the chosen framework's
  shared `AGENTS.md`.

### Adding as a submodule

```bash
git submodule add <REPO_URL> .agent/shared
```

#### Quarkus project

```bash
cd .agent/shared
git sparse-checkout init --cone
git sparse-checkout set java gradle quarkus
cd ../..

ln -s .agent/shared/quarkus/AGENTS.md AGENTS.md
ln -s .agent/shared/quarkus/AGENTS.md CLAUDE.md
```

#### Spring Boot project

```bash
cd .agent/shared
git sparse-checkout init --cone
git sparse-checkout set java gradle spring-boot
cd ../..

ln -s .agent/shared/spring-boot/AGENTS.md AGENTS.md
ln -s .agent/shared/spring-boot/AGENTS.md CLAUDE.md
```

### After cloning or repairing an existing setup

Initialise the submodule:

```bash
git submodule update --init --recursive
```

If `.agent/shared` already exists, rerun the framework-specific sparse checkout command to ensure
all required directories are present, then verify the root symlinks:

```bash
git -C .agent/shared sparse-checkout list
readlink AGENTS.md
readlink CLAUDE.md
```

Both `readlink` commands should resolve to `.agent/shared/<framework>/AGENTS.md`.

> **Note:** If a project needs project-specific changes to `AGENTS.md`, delete the root
> symlink(s), copy the file into the project root, then edit and commit it. If `CLAUDE.md`
> should continue mirroring the same instructions, recreate it as a symlink to the project-root
> `AGENTS.md`.

### After pulling

Use `git pull --rebase --recurse-submodules` for everyday pulls. It fetches the latest changes
and keeps the submodule working tree aligned with the pinned commit.

Use `git submodule update --remote --recursive` only when deliberately advancing the shared rules
to the latest commit. After doing that, commit and push the updated submodule pointer so teammates
pick up the bump on their next pull:

```bash
git submodule update --remote --recursive
git add .agent/shared
git commit -m "Bump agent-shared to latest"
git push
```

### Recommended aliases

Add these to `~/.gitconfig`:

```ini
[alias]
    pullr      = pull --rebase --recurse-submodules
    sub-update = submodule update --remote --recursive
```

`git pullr` replaces everyday `git pull` and keeps submodules in sync automatically.
`git sub-update` is a deliberate action: run it only when adopting new shared guidance, and always
follow it with a commit in the consumer project.
