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

### Using a coding agent for setup

For agent-assisted setup, use the dedicated guide in `docs/SETUP-CONSUMER-PROJECT.md`.
When using a hosted repository UI, copy the URL to that document and give it to the coding agent
together with the git URL of this repository.

Example prompt:

```text
Please follow the setup guide at 'https://github.com/yuriytkach/java-ai-common/blob/main/docs/SETUP-CONSUMER-PROJECT.md'
Connect java-ai-common to this repository using this submodule URL: https://github.com/yuriytkach/java-ai-common.git.
Detect whether this project is Quarkus or Spring Boot, verify the existing submodule setup or add it
if missing, configure the correct sparse checkout, create or fix the root AGENTS.md and CLAUDE.md
symlinks, and add/update the AI Agent Setup section in README.md.
Ask me only if the framework is ambiguous or if AGENTS.md or CLAUDE.md already exists as a regular file.
```

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
