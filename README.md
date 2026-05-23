# Agent Skills for Go HTTP Services

[![OpenSSF Scorecard](https://api.scorecard.dev/projects/github.com/openserbia/go-claude-skills/badge)](https://scorecard.dev/viewer/?uri=github.com/openserbia/go-claude-skills)

Opinionated AI agent skills for building production-grade Go HTTP services. Extracted from real production code — chi router, pgx, httpin, go-playground/validator, goose migrations, and clean layered architecture.

> Distilled from [openserbia/tg-statistic](https://github.com/openserbia/tg-statistic) production codebase. **Reviewed and curated by a human.**

## How to use

**Install with [skills](https://skills.sh/) CLI:**

```bash
# Install all skills
npx skills add openserbia/go-claude-skills -g -y

# Or a single skill
npx skills add openserbia/go-claude-skills@golang-service -g -y
```

<details>
<summary>Claude Code</summary>

```bash
npx skills add openserbia/go-claude-skills -g -y
```

</details>

<details>
<summary>Other agents (Cursor, Copilot, Gemini CLI, OpenClaw)</summary>

```bash
git clone https://github.com/openserbia/go-claude-skills.git ~/.agents/skills/go-claude-skills
```

</details>

## Skills

### Original

| Skill | Description |
| --- | --- |
| `golang-service` | Architecture and patterns — chi router, httpin binding, pgx, repository pattern (RW/RO split), error envelope, middleware, cursor-based pagination |
| `golang-tooling` | Build pipeline — golangci-lint v2, gofumpt + gci, Taskfile, multi-stage Docker with Devbox, goose migrations, testing conventions |
| `golang-scaffold` | Scaffold a new Go HTTP service with 22 infrastructure files from a single command |
| `golang-validation` | Request validation with go-playground/validator — complete tag reference, custom validators, cross-field validation, dive for collections, error humanization, httpin integration |
| `golang-github-actions` | GitHub Actions workflows — Devbox + Taskfile CI, Docker deploy, multi-service builds, auto-release, path filtering, self-hosted runners |

### Adapted from [samber/cc-skills-golang](https://github.com/samber/cc-skills-golang) (MIT)

| Skill | Description | Changes |
| --- | --- | --- |
| `golang-database` | Database access — parameterized queries, transactions, isolation levels, connection pools, migrations | Focused on pgx, replaced golang-migrate with goose, added RW/RO pool pattern |
| `golang-troubleshooting` | Systematic debugging — root cause analysis, pprof, race detection, Delve, production debugging (10 reference files) | Added goose migration debugging, aligned with slog |
| `golang-structs-interfaces` | Type design — composition, embedding, type assertions, interface segregation, dependency injection | Added httpin/validator tags, RW/RO repo interfaces, metrics decorator pattern |

## Stack

| Layer           | Tool                        |
| --------------- | --------------------------- |
| Router          | chi v5                      |
| Request binding | httpin                      |
| Validation      | go-playground/validator v10 |
| Database        | pgx v5 (raw SQL, no ORM)    |
| Migrations      | goose                       |
| Formatting      | gofumpt + gci               |
| Linting         | golangci-lint v2            |
| Build           | go-task (Taskfile)          |
| Docker          | Multi-stage Alpine + Devbox |
| Logging         | log/slog                    |
| Metrics         | Prometheus                  |

## Key patterns

- **Layered architecture** — `cmd/` -> `internal/config` -> `internal/db` -> `internal/domain` -> `internal/service` -> `internal/httpapi`
- **Repository pattern with RW/RO split** — separate read/write interfaces, separate connection pools
- **Metrics decorator** — wrap repositories with Prometheus timing without touching business logic
- **Typed request DTOs** — httpin for binding + validator for constraints, with `.ToDomain()` conversion
- **Unified error envelope** — consistent JSON errors with `error.code` (UPPER_SNAKE_CASE), `request_id`, `timestamp`
- **Cursor-based pagination** — base64-encoded offset, over-fetch by 1 for `has_more`
- **Two-layer validation** — tag-based field constraints + service-layer business rules
- **Graceful shutdown** — signal context with configurable timeout

## License

Original skills: MIT. Adapted skills retain their original MIT license from [samber/cc-skills-golang](https://github.com/samber/cc-skills-golang).
