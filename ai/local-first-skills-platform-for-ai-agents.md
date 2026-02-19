# Building a Local-First Skills Platform for AI Agents

AI agents are useful, but team workflows around them are often messy. Skills are stored in random folders, setup steps are manual, and security checks are inconsistent. This project solves that with one clear idea: keep skill management local, structured, and safe.

The system is built around a Go CLI, Docker-based execution, and Make targets. It supports Claude Code, Codex, and the Gemini CLI, and uses symlinks to share a single skill source across many projects.


## Why this project exists

Most teams face the same problems:

- Skill versions drift between machines.
- People copy skills from unknown places.
- There is no default audit before use.
- Every project repeats the same setup.

This project reduces those risks. You install once in a controlled repository, then mount it into other projects. No copy-paste sprawl. No hidden divergence.


## Core tech stack

The stack is intentionally simple:

- **Go** for the CLI (`skills` binary)
- **Docker** for controlled install/audit workflows
- **Make** for a stable command interface
- **Symlinks** for cross-project reuse
- **GitHub Actions + GHCR** for CI and release artifacts

This is a pragmatic stack. It favours reliability over novelty.


## Why Go is a strong fit

Go is a good choice here because the tool is mostly systems work:

- path handling
- file operations
- symlink management
- command execution
- predictable binaries

You get a fast startup, strong support for the standard library, and easy cross-platform builds. The project ships binaries for Linux/macOS and amd64/arm64, which makes adoption easier for mixed teams.


## Key workflows

The CLI and Make targets cover four main operations:

1. **Install all skills** (`make skills`)
2. **Add selected skills** (`make add-skill SKILLS='...'`)
3. **Mount skills into another project** (`make mount-skills TARGET_PROJECT=...`)
4. **Audit skills for security risks** (`make audit`)

These operations map directly to real team needs: setup, incremental updates, reuse, and trust.


## A smart reuse model: mount, don’t duplicate

The `mount-skills` command creates symlinks in target projects:

- `.agents -> source .agents`
- `.claude -> source .agents/skills`

This has two major advantages:

- **Single source of truth:** one update benefits all mounted projects.
- **Lower maintenance:** no repeated copies to keep in sync.

It also protects existing files by moving conflicts to timestamped backups before replacement. That is a practical safety feature teams need in production workflows.


## Security by default

Security is not optional in this design. The audit command runs a containerised scanner with behavioural checks and YARA rules. The default profile is strict and fails on findings.

Why this matters:

- Skills can include instructions that influence agent behaviour deeply.
- A bad skill can leak data or run unsafe actions.
- Automated scanning creates a baseline trust model before use.

This is a strong pattern: local execution, automated checks, and fail-fast defaults.


## Docker: benefits and honest tradeoffs

Docker brings reproducibility. Install and scan flows run with known toolchains, so behaviour is consistent across machines and CI.

The project also clearly documents a critical risk: workflows that mount `docker.sock` are effectively host-level privileged. It requires explicit opt-in via `ALLOW_DOCKER_SOCK=true`.

That transparency is important. The project does not hide risk; it makes risk explicit and controlled.


## Developer experience and operations

The repo gives multiple entry points:

- source build (`make build`)
- Docker image usage
- standalone binary from `GHCR`

This flexibility helps different environments: local dev, locked-down CI runners, and teams that prefer binaries only.

> Testing is also layered:

- unit
- functional
- e2e
- coverage gate (default minimum 80%)

That test split is a strong engineering choice. Unit tests catch logic issues early, while functional/e2e tests protect integration behaviour.


## CI/CD and release maturity

The release pipeline builds and publishes multi-arch artefacts for both manager and runner images, as well as standalone binaries. PR (pull requests) pipelines run lint, tests, and coverage checks.

This gives two operational advantages:

- safer merges
- predictable releases

For a shared internal tool, that maturity is a big advantage because breakage affects many downstream repos.


## Main reasons this approach is effective

- It is **local-first**: your code and prompts stay in your environment.
- It is **repeatable**: Make + Docker + Go reduce machine-specific drift.
- It is **secure by design**: audits are first-class, not an afterthought.
- It is **scalable across repos**: symlink mounting avoids duplication.
- It is **production-minded**: tests, coverage gates, and multi-arch release flow are built in.

## Final thoughts

This project is a solid example of practical engineering. It solves real workflow pain with clear boundaries and simple, durable tools. The design choices are not trendy. They are useful.

That is exactly what you want from infrastructure that supports AI-assisted development.
