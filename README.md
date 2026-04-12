# Go Claude Skills

Opinionated Claude Code skills for building Go HTTP services.

## Skills

### Original

| Skill | Description |
|-------|-------------|
| **golang-service** | Architecture and patterns — chi router, httpin binding, pgx, repository pattern, error handling, middleware, pagination |
| **golang-tooling** | Build pipeline, code quality — golangci-lint, gofumpt, Taskfile, Docker, goose migrations, testing conventions |
| **golang-scaffold** | Scaffold a new Go HTTP service with all infrastructure files from a single command |
| **golang-validation** | Request validation with go-playground/validator — tag reference, custom validators, cross-field, dive, error humanization |

### Adapted from [samber/cc-skills-golang](https://github.com/samber/cc-skills-golang) (MIT)

| Skill | Description | Changes |
|-------|-------------|---------|
| **golang-database** | Database access with pgx, parameterized queries, transactions, connection pools, migrations | Focused on pgx, replaced golang-migrate with goose, added RW/RO pool pattern |
| **golang-troubleshooting** | Systematic debugging — root cause analysis, pprof, race detection, Delve, production debugging | Added goose migration debugging, aligned with slog |
| **golang-structs-interfaces** | Struct and interface design — composition, embedding, type assertions, interface segregation, field tags | Added httpin/validator tags, RW/RO repo interfaces, metrics decorator pattern |

## Install

```bash
# Install all skills globally
npx skills add openserbia/go-claude-skills -g -y

# Or install individually
npx skills add openserbia/go-claude-skills@golang-service -g -y
npx skills add openserbia/go-claude-skills@golang-tooling -g -y
npx skills add openserbia/go-claude-skills@golang-scaffold -g -y
npx skills add openserbia/go-claude-skills@golang-validation -g -y
npx skills add openserbia/go-claude-skills@golang-database -g -y
npx skills add openserbia/go-claude-skills@golang-troubleshooting -g -y
npx skills add openserbia/go-claude-skills@golang-structs-interfaces -g -y
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
