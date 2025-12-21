# Go B2B Starter Backend

Professional Modular Monolith backend for B2B SaaS using idiomatic Go project layout.

## ⚡️ Quick Start

```bash
# 1. Start dependencies (Postgres, Redis)
cd deps && docker-compose up -d postgres redis

# 2. Copy environment config
cp example.env app.env

# 3. Run migrations
make migrateup

# 4. Start server with live reload
make dev
```

## 🏗 Project Layout (Go Standard 2026)

```
go-b2b-starter/
├── cmd/
│   └── api/              # Application entry point
│       └── main.go
│
├── internal/             # Private application code
│   ├── bootstrap/        # App initialization & DI wiring
│   ├── api/              # API route registration
│   │
│   ├── auth/             # Authentication & RBAC
│   ├── billing/          # Subscription & billing
│   ├── organizations/    # Multi-tenant org management
│   ├── documents/        # PDF document handling
│   ├── cognitive/        # AI/RAG chat features
│   │
│   ├── db/               # Database connections & SQLC
│   ├── server/           # HTTP server & middleware
│   ├── redis/            # Redis client
│   └── stytch/           # Stytch B2B auth adapter
│
├── pkg/                  # Public reusable packages
│   ├── httperr/          # HTTP error types
│   ├── pagination/       # Pagination helpers
│   ├── response/         # API response helpers
│   └── slugify/          # String utilities
│
├── deps/                 # Docker Compose for dependencies
├── docs/                 # Documentation
└── go.mod                # Single module (consolidated)
```

## 📚 Documentation

- **[Architecture Guide](./docs/01-architecture.md)** - Understand the layers
- **[Adding a Feature](./docs/02-adding-a-module.md)** - How to create new features
- **[API & Auth](./docs/03-api-and-auth.md)** - Security and Request flow

## 🛠 Key Commands

| Command | Description |
|---------|-------------|
| `make dev` | Start server with Air (Live Reload) |
| `make server` | Run server directly |
| `make build` | Build binary to `bin/api` |
| `make migrateup` | Apply DB migrations |
| `make sqlc` | Generate type-safe DB code |
| `make swagger` | Generate Swagger docs |

## 🔧 Module Structure

Each feature module in `internal/` follows **Clean Architecture**:

```
internal/billing/
├── cmd/              # Module initialization (DI)
│   └── init.go
├── app/              # Application layer (use cases)
│   └── services/
├── domain/           # Core business logic & interfaces
├── infra/            # External integrations
│   └── repositories/
├── handler.go        # HTTP handlers
├── routes.go         # Route registration
└── provider.go       # Dependency injection
```

## 🚀 API Endpoints

The server exposes these API groups:

- `/api/auth/*` - Authentication & member management
- `/api/organizations/*` - Organization CRUD
- `/api/accounts/*` - Account management
- `/api/rbac/*` - Role & permission discovery
- `/api/subscriptions/*` - Billing status
- `/api/example_documents/*` - PDF upload/management
- `/api/example_cognitive/*` - AI chat sessions
- `/swagger/*` - API documentation
- `/health` - Health check
