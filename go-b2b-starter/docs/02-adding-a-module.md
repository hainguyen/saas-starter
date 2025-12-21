# Adding a New Module

This guide shows how to create a new feature module following Clean Architecture and the idiomatic Go project layout.

## Module Location

All feature modules live in `internal/` which enforces Go's import boundary:

```
internal/
├── auth/             # Authentication & RBAC
├── billing/          # Subscription & billing
├── organizations/    # Multi-tenant organizations
├── documents/        # Document management
├── cognitive/        # AI/RAG features
└── projects/         # ← Your new module here
```

## Module Structure

Each module follows **Clean Architecture** with these layers:

```
internal/projects/
├── cmd/                      # Module initialization (DI wiring)
│   └── init.go
│
├── app/                      # Application Layer (Use Cases)
│   └── services/
│       └── project_service.go
│
├── domain/                   # Domain Layer (Core Business Logic)
│   ├── entities.go           # Data structures
│   └── repository.go         # Interface definitions
│
├── infra/                    # Infrastructure Layer (External)
│   └── repositories/
│       └── postgres_repository.go
│
├── handler.go                # HTTP handlers (Delivery Layer)
├── routes.go                 # Route registration
└── provider.go               # Dependency injection setup
```

## Step-by-Step Guide

### 1. Define the Entity (`domain/entities.go`)

Start with your core business objects:

```go
package domain

import "time"

type Project struct {
    ID             int32     `json:"id"`
    Name           string    `json:"name"`
    Description    string    `json:"description"`
    OrganizationID int32     `json:"organization_id"`
    CreatedAt      time.Time `json:"created_at"`
    UpdatedAt      time.Time `json:"updated_at"`
}
```

### 2. Define the Repository Interface (`domain/repository.go`)

Define what operations your module needs:

```go
package domain

import "context"

type ProjectRepository interface {
    Create(ctx context.Context, p *Project) error
    GetByID(ctx context.Context, id int32) (*Project, error)
    ListByOrganization(ctx context.Context, orgID int32) ([]*Project, error)
    Update(ctx context.Context, p *Project) error
    Delete(ctx context.Context, id int32) error
}
```

### 3. Implement the Repository (`infra/repositories/postgres_repository.go`)

Implement the interface using your database:

```go
package repositories

import (
    "context"

    "go-b2b-starter/internal/db"
    "go-b2b-starter/internal/projects/domain"
)

type PostgresProjectRepository struct {
    queries *db.Queries
}

func NewPostgresProjectRepository(queries *db.Queries) domain.ProjectRepository {
    return &PostgresProjectRepository{queries: queries}
}

func (r *PostgresProjectRepository) Create(ctx context.Context, p *domain.Project) error {
    result, err := r.queries.CreateProject(ctx, db.CreateProjectParams{
        Name:           p.Name,
        Description:    p.Description,
        OrganizationID: p.OrganizationID,
    })
    if err != nil {
        return err
    }
    p.ID = result.ID
    return nil
}

// ... implement other methods
```

### 4. Create the Service (`app/services/project_service.go`)

Business logic lives here:

```go
package services

import (
    "context"

    "go-b2b-starter/internal/projects/domain"
)

type ProjectService interface {
    Create(ctx context.Context, orgID int32, name, description string) (*domain.Project, error)
    GetByID(ctx context.Context, id int32) (*domain.Project, error)
    ListByOrganization(ctx context.Context, orgID int32) ([]*domain.Project, error)
}

type projectService struct {
    repo domain.ProjectRepository
}

func NewProjectService(repo domain.ProjectRepository) ProjectService {
    return &projectService{repo: repo}
}

func (s *projectService) Create(ctx context.Context, orgID int32, name, description string) (*domain.Project, error) {
    project := &domain.Project{
        Name:           name,
        Description:    description,
        OrganizationID: orgID,
    }

    if err := s.repo.Create(ctx, project); err != nil {
        return nil, err
    }

    return project, nil
}
```

### 5. Create the Handler (`handler.go`)

HTTP request handling:

```go
package projects

import (
    "net/http"

    "github.com/gin-gonic/gin"

    "go-b2b-starter/internal/auth"
    "go-b2b-starter/internal/projects/app/services"
    "go-b2b-starter/pkg/response"
)

type Handler struct {
    service services.ProjectService
}

func NewHandler(service services.ProjectService) *Handler {
    return &Handler{service: service}
}

type CreateProjectRequest struct {
    Name        string `json:"name" binding:"required"`
    Description string `json:"description"`
}

func (h *Handler) Create(c *gin.Context) {
    ctx := auth.GetRequestContext(c)

    var req CreateProjectRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        response.BadRequest(c, err.Error())
        return
    }

    project, err := h.service.Create(c.Request.Context(), ctx.OrganizationID, req.Name, req.Description)
    if err != nil {
        response.InternalError(c, err)
        return
    }

    c.JSON(http.StatusCreated, project)
}
```

### 6. Define Routes (`routes.go`)

Register your endpoints:

```go
package projects

import (
    "github.com/gin-gonic/gin"

    "go-b2b-starter/internal/auth"
)

func RegisterRoutes(router *gin.RouterGroup, handler *Handler, authMiddleware *auth.Middleware) {
    projects := router.Group("/projects")
    projects.Use(authMiddleware.Authenticate())
    {
        projects.POST("", handler.Create)
        projects.GET("/:id", handler.GetByID)
        projects.GET("", handler.List)
        projects.PUT("/:id", handler.Update)
        projects.DELETE("/:id", handler.Delete)
    }
}
```

### 7. Wire Dependencies (`cmd/init.go`)

Set up dependency injection:

```go
package cmd

import (
    "go.uber.org/dig"

    "go-b2b-starter/internal/db"
    "go-b2b-starter/internal/projects"
    "go-b2b-starter/internal/projects/app/services"
    "go-b2b-starter/internal/projects/domain"
    "go-b2b-starter/internal/projects/infra/repositories"
)

func InitModule(container *dig.Container) error {
    // Repository
    if err := container.Provide(func(queries *db.Queries) domain.ProjectRepository {
        return repositories.NewPostgresProjectRepository(queries)
    }); err != nil {
        return err
    }

    // Service
    if err := container.Provide(services.NewProjectService); err != nil {
        return err
    }

    // Handler
    if err := container.Provide(projects.NewHandler); err != nil {
        return err
    }

    return nil
}
```

### 8. Register in Bootstrap

Add your module to `internal/bootstrap/init_mods.go`:

```go
import projectsCmd "go-b2b-starter/internal/projects/cmd"

func InitMods(container *dig.Container) error {
    // ... existing modules ...

    // Projects module
    if err := projectsCmd.InitModule(container); err != nil {
        return err
    }

    return nil
}
```

### 9. Register Routes in API

Add routes in `internal/api/routes.go`:

```go
import "go-b2b-starter/internal/projects"

func RegisterRoutes(router *gin.Engine, ..., projectHandler *projects.Handler) {
    api := router.Group("/api")

    // ... existing routes ...

    projects.RegisterRoutes(api, projectHandler, authMiddleware)
}
```

## Database Migration

Create a migration for your new table:

```bash
make createmigration name=create_projects_table
```

Edit the generated migration file:

```sql
-- +goose Up
CREATE TABLE projects (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    organization_id INTEGER NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_projects_organization_id ON projects(organization_id);

-- +goose Down
DROP TABLE IF EXISTS projects;
```

Then run:

```bash
make migrateup
make sqlc  # Regenerate type-safe queries
```

## Best Practices

1. **Domain Layer is Pure**: No external dependencies in `domain/`
2. **Interfaces in Domain**: Repository interfaces defined where they're used
3. **Services Return Domain Types**: Not database types
4. **Handlers Are Thin**: Validation, auth, delegate to service
5. **Context First**: Always pass `context.Context` as first parameter
