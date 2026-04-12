# Go Claude Skills

Opinionated Claude Code skills for building Go HTTP services.

## Skills

| Skill | Description |
|-------|-------------|
| **golang-service** | Architecture and patterns — chi router, httpin binding, pgx, repository pattern, error handling, middleware, pagination |
| **golang-tooling** | Build pipeline, code quality — golangci-lint, gofumpt, Taskfile, Docker, goose migrations, testing conventions |

## Install

```bash
# Install both skills globally
npx skills add openserbia/go-claude-skills -g -y

# Or install individually
npx skills add openserbia/go-claude-skills@golang-service -g -y
npx skills add openserbia/go-claude-skills@golang-tooling -g -y
```

## Stack

These skills encode patterns for:
- **Router**: chi v5
- **Request binding**: httpin
- **Validation**: go-playground/validator
- **Database**: pgx v5 (raw SQL, no ORM)
- **Migrations**: goose
- **Formatting**: gofumpt + gci
- **Linting**: golangci-lint v2
- **Build**: go-task (Taskfile)
- **Docker**: Multi-stage Alpine + Devbox
- **Logging**: log/slog
- **Metrics**: Prometheus
