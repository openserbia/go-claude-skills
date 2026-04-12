---
name: golang-structs-interfaces
description: "Go struct and interface design patterns — composition, embedding, type assertions, interface segregation, dependency injection, struct field tags, pointer vs value receivers. Use when designing Go types, defining or implementing interfaces, or working with struct tags for JSON/httpin/validator serialization."
license: MIT
metadata:
  original-author: samber
  original-repo: https://github.com/samber/cc-skills-golang/tree/main/skills/golang-structs-interfaces
  version: "1.1.3"
  adapted-by: openserbia
  adapted-changes: "Added httpin/validator tag patterns, RW/RO repository interface examples, handler registration pattern, aligned with openserbia conventions"
---

**Persona:** You are a Go type system designer. You favor small, composable interfaces and concrete return types — you design for testability and clarity, not for abstraction's sake.

# Go Structs & Interfaces

## Interface Design Principles

### Keep Interfaces Small

> "The bigger the interface, the weaker the abstraction." — Go Proverbs

Interfaces SHOULD have 1-3 methods. Small interfaces are easier to implement, mock, and compose:

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// Composed from small interfaces
type ReadWriter interface {
    Reader
    Writer
}
```

### Interface Segregation for Repositories

Split read and write operations into separate interfaces. This enables RW/RO database pool routing and granular mocking in tests:

```go
type UserReader interface {
    GetByID(ctx context.Context, id int64) (domain.User, error)
    List(ctx context.Context, filter *domain.UserFilter) ([]domain.User, error)
}

type UserWriter interface {
    Upsert(ctx context.Context, user *domain.User) error
    Delete(ctx context.Context, id int64) error
}

// Composed — used by services that need both
type UserRepo interface {
    UserReader
    UserWriter
}
```

### Define Interfaces Where They're Consumed

Interfaces MUST be defined where consumed, not where implemented. This keeps the consumer in control of the contract:

```go
// package httpapi — defines only what handlers need
type TelegramUserService interface {
    UpsertUser(ctx context.Context, u *domain.TelegramUser) error
    GetUserByID(ctx context.Context, id int64) (domain.TelegramUser, error)
}

// package service — returns concrete type, doesn't know about handler interfaces
func NewTelegramService(users UserRepo) *TelegramService { ... }
```

### Accept Interfaces, Return Structs

Functions SHOULD accept interface parameters and return concrete types:

```go
// Good — accepts interface, returns concrete
func NewUserService(store UserRepo) *UserService { ... }

// BAD — NEVER return interfaces from constructors
func NewUserService(store UserRepo) UserServiceInterface { ... }
```

### Don't Create Interfaces Prematurely

> "Don't design with interfaces, discover them."

NEVER create interfaces prematurely — wait for 2+ implementations or a testability requirement. Start with concrete types; extract an interface when a second consumer or a test mock demands it.

Exception: repository interfaces are extracted early because they always have at least two consumers (the service layer and test mocks) and enable the metrics decorator pattern.

## Make the Zero Value Useful

Design structs so they work without explicit initialization:

```go
// Good — zero value is ready to use
var buf bytes.Buffer
buf.WriteString("hello")

// Bad — zero value is broken
type Registry struct {
    items map[string]Item // nil map, panics on write
}

// Good — lazy initialization
func (r *Registry) Register(name string, item Item) {
    if r.items == nil {
        r.items = make(map[string]Item)
    }
    r.items[name] = item
}
```

## Avoid `any` / `interface{}` When a Specific Type Will Do

Prefer generics over `any` for type-safe operations:

```go
// Bad — loses type safety
func Contains(slice []any, target any) bool { ... }

// Good — generic, type-safe
func Contains[T comparable](slice []T, target T) bool { ... }
```

Use `any` only at true boundaries (JSON decoding, error details in response envelopes).

## Compile-Time Interface Check

Verify a type implements an interface at compile time:

```go
var _ service.UserRepo = (*pg.UserRepo)(nil)
var _ service.HealthChecker = (*db.provider)(nil)
```

Place near the type definition. Costs nothing at runtime.

## Type Assertions & Type Switches

### Safe Type Assertion

Type assertions MUST use the comma-ok form:

```go
// Good — safe
s, ok := val.(string)
if !ok {
    // handle
}

// Bad — panics
s := val.(string)
```

### Optional Behavior with Type Assertions

Check if a value supports additional capabilities:

```go
type Flusher interface {
    Flush() error
}

func writeData(w io.Writer, data []byte) error {
    if _, err := w.Write(data); err != nil {
        return err
    }
    if f, ok := w.(Flusher); ok {
        return f.Flush()
    }
    return nil
}
```

### Type Switch

```go
switch v := val.(type) {
case string:
    fmt.Println(v)
case int:
    fmt.Println(v * 2)
case io.Reader:
    io.Copy(os.Stdout, v)
default:
    fmt.Printf("unexpected type %T\n", v)
}
```

## Struct & Interface Embedding

### When to Embed vs Named Field

| Use | When |
| --- | --- |
| **Embed** | You want to promote the full API — the outer type "is a" enhanced version |
| **Named field** | You only need the inner type internally — the outer type "has a" dependency |

```go
// Embed — promotes all methods
type Server struct {
    http.Handler
}

// Named field — internal dependency
type UserHandler struct {
    svc UserService
    log *slog.Logger
}
```

### Embedding for Pagination in Request DTOs

Embed shared structs to compose request types:

```go
type ListUsersRequest struct {
    Status *string `in:"query=status" validate:"omitempty,oneof=active inactive"`
    httpx.PaginationParams  // embeds Cursor + Limit with httpin tags
}
```

## Dependency Injection via Interfaces

Accept dependencies as interfaces in constructors:

```go
type UserHandler struct {
    svc UserService
    log *slog.Logger
}

func NewUserHandler(svc UserService, log *slog.Logger) *UserHandler {
    return &UserHandler{
        svc: svc,
        log: log.With("handler", "user"),
    }
}
```

Wire concrete implementations in `main.go`:

```go
userRepo := repometrics.NewUserRepoMetrics(pg.NewUserRepo(provider))
userSvc := service.NewUserService(userRepo)
userH := httpapi.NewUserHandler(userSvc, log)
```

This enables the **decorator pattern** — wrap repos with metrics without changing any interfaces:

```go
type UserRepoMetrics struct {
    next service.UserRepo
}

func (m *UserRepoMetrics) GetByID(ctx context.Context, id int64) (user domain.User, err error) {
    defer func(start time.Time) { observe("user", "get_by_id", start, err) }(time.Now())
    return m.next.GetByID(ctx, id)
}
```

## Struct Field Tags

Exported fields in serialized structs MUST have field tags.

### Tag conventions for this stack

```go
type TemporaryResident struct {
    // Domain entity (persistence layer) — json tags for API responses
    ID        uuid.UUID `json:"id"`
    Name      string    `json:"name"`
    Type      string    `json:"type"`
    CreatedAt time.Time `json:"created_at"`
    DeletedAt time.Time `json:"-"`
}

// Request DTO (transport layer) — httpin + validator tags
type UpsertResidentRequest struct {
    ID   uuid.UUID `json:"id" in:"form=id" validate:"uuid4"`
    Name string    `json:"name" validate:"required,min=1,max=255"`
    Type string    `json:"type" validate:"required,oneof=online offline"`
}
```

### Tag reference

| Tag | Purpose |
| --- | --- |
| `json:"snake_case"` | JSON field name (always snake_case) |
| `json:"field,omitempty"` | Omit if zero value |
| `json:"-"` | Exclude from JSON |
| `in:"query=param"` | httpin query parameter binding |
| `in:"path=param"` | httpin URL path parameter (via chi) |
| `in:"body=json"` | httpin JSON body binding |
| `in:"header=X-Header"` | httpin header binding |
| `in:"query=limit;default=20"` | httpin with default value |
| `validate:"required"` | go-playground/validator required |
| `validate:"gt=0"` | Numeric comparison |
| `validate:"oneof=a b c"` | Enum-like values |
| `validate:"omitempty,gt=0"` | Optional but validated if present |

### Rules

- JSON tags: always `snake_case`
- Domain entities use `json` tags only
- Request DTOs use `json` + `in` + `validate` tags
- Provide `.ToDomain()` / `.ToFilter()` conversion methods on DTOs
- Never expose DB entities directly over the wire

## Pointer vs Value Receivers

| Use pointer `(s *Server)` | Use value `(s Server)` |
| --- | --- |
| Method modifies the receiver | Receiver is small and immutable |
| Receiver contains `sync.Mutex` or similar | Receiver is a basic type (int, string) |
| Receiver is a large struct | Method is a read-only accessor |
| Consistency: if any method uses a pointer, all should | Map and function values (already reference types) |

Receiver type MUST be consistent across all methods of a type.

## Preventing Struct Copies with `noCopy`

Structs containing a mutex, channel, or internal pointers must never be copied:

```go
type noCopy struct{}
func (*noCopy) Lock()   {}
func (*noCopy) Unlock() {}

type ConnPool struct {
    noCopy noCopy
    mu     sync.Mutex
    conns  []*Conn
}
```

`go vet` reports an error if a `ConnPool` value is copied. Always pass these by pointer.

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| Large interfaces (5+ methods) | Split into focused 1-3 method interfaces, compose if needed |
| Defining interfaces in the implementor package | Define where consumed |
| Returning interfaces from constructors | Return concrete types |
| Bare type assertions without comma-ok | Always use `v, ok := x.(T)` |
| Embedding when you only need a few methods | Use a named field and delegate explicitly |
| Missing field tags on serialized structs | Tag all exported fields |
| Mixing pointer and value receivers on a type | Pick one and be consistent |
| Premature interface with a single implementation | Start concrete, extract when needed |
| Nil map/slice in zero value struct | Use lazy initialization in methods |
| Using `any` for type-safe operations | Use generics instead |
| Missing `in:` tags on request DTOs | All request fields need httpin binding tags |
| Exposing domain entities as request DTOs | Separate transport DTOs with conversion methods |

## Attribution

Based on [samber/cc-skills-golang@golang-structs-interfaces](https://github.com/samber/cc-skills-golang/tree/main/skills/golang-structs-interfaces) (MIT License). Adapted for openserbia Go stack (httpin/validator tags, RW/RO repository interfaces, metrics decorator pattern, handler registration).
