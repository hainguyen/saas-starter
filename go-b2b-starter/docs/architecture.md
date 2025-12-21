# Architecture Guide

The codebase uses Clean Architecture with dependency injection to maintain separation of concerns and testability.

## Project Layout (Go Standard 2026)

```
go-b2b-starter/
├── cmd/                  # Application entry points
│   └── api/
│       └── main.go       # Main entry point
│
├── internal/             # Private application code (import boundary)
│   ├── bootstrap/        # Application initialization
│   ├── api/              # Route registration
│   │
│   ├── auth/             # Authentication & RBAC module
│   ├── billing/          # Subscription & billing module
│   ├── organizations/    # Multi-tenant organizations
│   ├── documents/        # Document management
│   ├── cognitive/        # AI/RAG features
│   │
│   ├── db/               # Database layer (SQLC)
│   ├── server/           # HTTP server & middleware
│   ├── redis/            # Redis client
│   └── stytch/           # Auth provider adapter
│
├── pkg/                  # Public reusable packages
│   ├── httperr/          # HTTP error types
│   ├── pagination/       # Pagination helpers
│   └── response/         # API response utilities
│
└── go.mod                # Single consolidated module
```

## Clean Architecture Layers

The project is organized into four distinct layers:

**1. Domain Layer** - Business entities and rules (innermost)
**2. Application Layer** - Use cases and business logic
**3. Infrastructure Layer** - External services and data access
**4. API Layer** - HTTP handlers and routes (outermost)

### Dependency Flow

Dependencies point **inward only**:

```
API → Application → Domain ← Infrastructure
```

- Domain layer has zero external dependencies
- Infrastructure implements domain interfaces
- Outer layers depend on inner layers, never the reverse

## Layer Responsibilities

### Domain Layer (`internal/{module}/domain/`)

The core business logic layer.

**Contains:**
- Entities with business rules
- Repository interfaces (contracts)
- Domain errors
- Validation logic

**Key principle**: No external dependencies. Pure business logic only.

### Application Layer (`internal/{module}/app/`)

Orchestrates domain operations to implement use cases.

**Contains:**
- Service interfaces and implementations
- Request/response types
- Transaction boundaries
- Business workflow coordination

**Key principle**: Uses domain interfaces, never infrastructure directly.

### Infrastructure Layer (`internal/{module}/infra/`)

Implements domain interfaces using concrete technologies.

**Contains:**
- Repository implementations
- Database adapters
- External service clients
- Type conversions (domain ↔ database)

**Key principle**: Depends on domain interfaces. Hidden behind abstractions.

### Handler Layer (`internal/{module}/handler.go`)

Handles HTTP concerns.

**Contains:**
- HTTP handlers
- Route definitions
- Request validation
- Response formatting

**Key principle**: Thin layer that delegates to application services.

## Dependency Injection

Uses [uber-go/dig](https://github.com/uber-go/dig) for automatic dependency injection.

### Core Pattern

```go
// 1. Define interface in domain
type ResourceRepository interface {
    GetByID(ctx context.Context, id int32) (*Resource, error)
}

// 2. Implement in infrastructure
type resourceRepository struct {
    queries *db.Queries
}

// 3. Register in DI container
container.Provide(func(queries *db.Queries) domain.ResourceRepository {
    return NewResourceRepository(queries)
})

// 4. Inject into services
container.Provide(func(repo domain.ResourceRepository) services.ResourceService {
    return services.NewResourceService(repo)
})
```

### Benefits

- Automatic dependency resolution
- Easy testing with mocks
- Clear dependency graph
- No manual wiring

## Module Pattern

Each business module follows a standard structure:

```
internal/{module}/
├── domain/           # Entities, interfaces
├── app/services/     # Business logic
├── infra/            # Implementations
├── cmd/init.go       # Initialization
├── handler.go        # HTTP handlers
├── routes.go         # Route registration
└── provider.go       # DI registration
```

### Module Registration

Every module has initialization in `cmd/init.go` that registers its dependencies:

- Repositories (infrastructure → domain interface)
- Services (application layer)
- Handlers (HTTP layer)

### Initialization Order

Defined in `internal/bootstrap/init_mods.go`:

1. **Infrastructure** - Database, logging, server
2. **Shared Services** - File storage, event bus, payments
3. **Authentication** - Redis, Stytch, auth middleware
4. **Domain Modules** - Organizations, billing, etc.
5. **API Layer** - Route registration

**Why order matters**: Each phase depends on previous phases being initialized.

## Resolver Pattern

Bridges authentication with domain modules without creating circular dependencies.

### Problem

Auth middleware needs to convert provider IDs (Stytch) to database IDs, but can't depend on domain modules directly.

### Solution

Define minimal interfaces in auth package:

```go
// Auth defines what it needs
type OrganizationResolver interface {
    ResolveByProviderID(ctx context.Context, providerID string) (int32, error)
}
```

Domain modules implement via adapters:

```go
// Module provides implementation
type orgResolverAdapter struct {
    repo domain.ResourceRepository
}
```

Wired together in `init_mods.go` during initialization.

## Best Practices

### Constructor Pattern

Always return interfaces, not concrete types:

```go
// ✅ Good
func NewResourceService(repo domain.ResourceRepository) services.ResourceService {
    return &resourceService{repo: repo}
}

// ❌ Bad
func NewResourceService(repo domain.ResourceRepository) *resourceService {
    return &resourceService{repo: repo}
}
```

### Context Handling

Context is always the first parameter:

```go
func (s *service) CreateResource(ctx context.Context, req *Request) error
```

### Error Wrapping

Add context to errors before returning:

```go
if err := s.repo.Create(ctx, resource); err != nil {
    return fmt.Errorf("failed to create resource: %w", err)
}
```

### Explicit Dependencies

All dependencies through constructor parameters:

```go
func NewResourceService(
    repo domain.ResourceRepository,
    eventBus eventbus.EventBus,
    logger logger.Logger,
) services.ResourceService
```

Never use global variables or hidden dependencies.

## File Locations

| Pattern | File |
|---------|------|
| DI container setup | `internal/bootstrap/root.go` |
| Module initialization order | `internal/bootstrap/init_mods.go` |
| Module DI registration | `internal/*/cmd/init.go` |
| Package initialization | `pkg/*/init.go` |
| API route setup | `internal/api/routes.go` |

## Next Steps

- **Database operations**: See [Database Guide](./database.md)
- **Authentication setup**: See [Authentication Guide](./authentication.md)
- **Building APIs**: See [API Development Guide](./api-development.md)
