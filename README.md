# ⭐ Production SaaS Starter Kit

The Enterprise-Grade SaaS boilerplate for serious founders. Built with **Next.js 16** and **Go 1.25**.

[![Go Report Card](https://goreportcard.com/badge/github.com/moasq/production-saas-starter)](https://goreportcard.com/report/github.com/moasq/production-saas-starter)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)

![Dashboard Preview](docs/dashboard.png)

## 🛠️ Built With

### Frontend Stack

- **[Next.js 16](https://nextjs.org)** (v16.0.10)
  Modern React framework with App Router and API routes.
- **[React 19](https://react.dev)** (v19.2.3)
  Latest React with improved performance and concurrent features.
- **[TypeScript](https://www.typescriptlang.org)** (v5.7.3)
  Type-safe JavaScript for enhanced developer experience.
- **[Tailwind CSS](https://tailwindcss.com)** (v3.4.17)
  Utility-first CSS framework for rapid UI development.
- **[shadcn/ui](https://ui.shadcn.com)** + **Radix UI**
  Accessible component library with 29+ pre-built components.
- **[TanStack Query](https://tanstack.com/query)** (v5.90.5)
  Powerful data fetching and state management.
- **[Zustand](https://zustand-demo.pmnd.rs)** (v5.0.8)
  Lightweight state management for UI state.
- **[react-hook-form](https://react-hook-form.com)** + **[Zod](https://zod.dev)**
  Type-safe forms with schema validation.
- **[Stytch](https://stytch.com)**
  Enterprise authentication with magic links, OAuth, and SSO.
- **[Polar.sh](https://polar.sh)**
  Billing integration and subscription management.
- **[Recharts](https://recharts.org)**
  Composable charting library for data visualization.

### Backend Stack

- **[Go 1.25](https://go.dev)**
  High-performance, concurrent backend with excellent tooling.
- **[Gin](https://gin-gonic.com)**
  Fast HTTP web framework with middleware support.
- **[PostgreSQL](https://www.postgresql.org)** with **[pgvector](https://github.com/pgvector/pgvector)**
  Reliable relational database with vector similarity search.
- **[SQLC](https://sqlc.dev)**
  Type-safe SQL compiler for Go (no ORM).
- **[Stytch B2B](https://stytch.com)**
  Enterprise authentication, SSO, and RBAC.
- **[Polar.sh](https://polar.sh)**
  Merchant of Record for subscriptions, invoicing, and global tax compliance.
- **[OpenAI API](https://openai.com)**
  LLM integration with RAG pipeline and vector embeddings.
- **[Mistral AI](https://mistral.ai)**
  OCR service for document data extraction.
- **[Cloudflare R2](https://www.cloudflare.com/products/r2/)**
  Object storage for file management.
- **[Docker](https://www.docker.com)** + **Docker Compose**
  Containerization for consistent environments.

## 🥇 Features

- **Authentication**: Sign in with Magic Link, Google OAuth, and Enterprise SSO.
- **Multi-Tenancy**: Built-in Organization support with strict data isolation.
- **Roles & Permissions**: Granular RBAC system with 3 roles (Member, Manager, Admin) and 7 permission types.
- **Billing & Subscriptions**: Complete integration with Polar.sh for SaaS pricing models.
- **AI & RAG**: Ready-to-use vector embeddings pipeline for AI features.
- **OCR Service**: Extract structured data from valid documents instantly.
- **Team Management**: Invite members, manage roles, and update settings.
- **Responsive Design**: Mobile-first UI built with Tailwind CSS and shadcn/ui.
- **Type Safety**: End-to-end type safety from database (SQLC) to frontend (TypeScript).

## ➡️ Coming Soon

- **Audit Logs**: Complete audit logging system for tracking user activities.
- **Webhooks UI**: Customer-facing webhook configuration.
- **Advanced Analytics**: Built-in charts and usage tracking.

## ✨ Getting Started

Please follow these simple steps to get a local copy up and running.

### Prerequisites

- **Docker** & **Docker Compose**
- **Go 1.25+**
- **Node.js 20+** & **pnpm**

### The One-Line Setup

Run this command to configure your keys and start the infrastructure:

```bash
chmod +x setup.sh && ./setup.sh
```

**Manual Start:**

1.  **Backend:** `cd go-b2b-starter && make dev`
2.  **Frontend:** `cd next_b2b_starter && pnpm dev`
3.  **Visit:** [http://localhost:3000](http://localhost:3000)

> [!IMPORTANT]
> See **[SETUP.md](./SETUP.md)** for full details.

## 🛡️ License

[MIT License](./LICENSE)

## 👯 Consulting & Services

Although this kit is self-service, I help ambitious founders move faster.

**I can help you with:**
1.  **Managed Config:** I sets up your AWS/GCP production environment so you never touch DevOps.
2.  **Custom Features:** Need SAML SSO or complex RAG flows? I'll build them directly into your repo.
3.  **Code Audits:** Migrating from Node/Python? I'll review your architecture for scale.

**[m.salim@apflowhq.com](mailto:m.salim@apflowhq.com)** • [**@foundmod**](https://x.com/foundmod)
