# Go Claude Skills

Opinionated Claude Code skills for building Go HTTP services.

## Skills

| Skill | Description | Origin |
|-------|-------------|--------|
| **golang-service** | Architecture and patterns — chi router, httpin binding, pgx, repository pattern, error handling, middleware, pagination | Original |
| **golang-tooling** | Build pipeline, code quality — golangci-lint, gofumpt, Taskfile, Docker, goose migrations, testing conventions | Original |
| **golang-scaffold** | Scaffold a new Go HTTP service with all infrastructure files from a single command | Original |
| **golang-database** | Database access with pgx, parameterized queries, transactions, connection pools, goose migrations | Adapted from [samber/cc-skills-golang](https://github.com/samber/cc-skills-golang/tree/main/skills/golang-database) (MIT) |
| **golang-troubleshooting** | Systematic debugging — root cause analysis, pprof, race detection, Delve, production debugging | Adapted from [samber/cc-skills-golang](https://github.com/samber/cc-skills-golang/tree/main/skills/golang-troubleshooting) (MIT) |
| **golang-structs-interfaces** | Struct and interface design — composition, embedding, type assertions, interface segregation, field tags | Adapted from [samber/cc-skills-golang](https://github.com/samber/cc-skills-golang/tree/main/skills/golang-structs-interfaces) (MIT) |
| **golang-validation** | Request validation with go-playground/validator — tag reference, custom validators, cross-field, dive, error humanization | Original |

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
