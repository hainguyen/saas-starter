# B2B SaaS Starter
 
 A modern B2B SaaS starter kit with Authentication, Billing, and RBAC.
 
 ## 🚀 Quick Start
 
 ```bash
 ./setup.sh
 cd next_b2b_starter
 pnpm dev
 ```
 
 Visit `http://localhost:3000`.
 
 ## 📚 Documentation
 
 - **[Getting Started](./docs/01-getting-started.md)** - Setup and installation
 - **[Authentication](./docs/02-authentication.md)** - How auth works
 - **[Permissions](./docs/03-permissions-and-roles.md)** - RBAC system
 - **[Payments](./docs/04-payments-and-billing.md)** - Subscriptions
 - **[Full Documentation](./docs/README.md)** - Complete guide index
 
 ## 🔧 Advanced
 
 - **[Stytch Configuration](./STYTCH_CONFIGURATION.md)** - Advanced auth security settings
 - **[Claude Guide](./CLAUDE.md)** - Development rules
 
 ## Features
 
 - **Stack**: Next.js 16, TypeScript, Tailwind, shadcn/ui
 - **Auth**: Stytch B2B (passwordless)
 - **Billing**: Polar.sh integration
 - **State**: React Query + Zustand
 
 ## Project Structure
 
 - `app/`: Next.js App Router
 - `components/`: UI Components
 - `lib/`: Business logic, hooks, API clients
 - `middleware.ts`: Auth protection
 
 ## Support
 
 Open an issue on GitHub for support. License: MIT.
