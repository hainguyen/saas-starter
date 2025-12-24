# Documentation Index
 
 Welcome to the B2B SaaS Starter documentation.
 
 ## What is B2B SaaS Starter?
 
 B2B SaaS Starter is a B2B SaaS application. It's built with Next.js 16, TypeScript, and modern best practices.
 
 ## Tech Stack
 
 - **Framework**: Next.js 16 with App Router
 - **Language**: TypeScript (strict mode)
 - **Styling**: Tailwind CSS + shadcn/ui components
 - **State**: Zustand for global state
 - **Data Fetching**: TanStack React Query
 - **Authentication**: Stytch B2B
 - **Payments**: Stripe/Polar integration
 
 ## Core Guides
 
 1. **[Getting Started](./01-getting-started.md)**
    Setup, installation, and first run.
 
 2. **[Authentication](./02-authentication.md)**
    Protecting routes and checking login status.
 
 3. **[Permissions & Roles](./03-permissions-and-roles.md)**
    Using RBAC (Admin, Manager, Member) and permissions.
 
 4. **[Payments & Billing](./04-payments-and-billing.md)**
    Checking subscription status and paywalls.
 
 5. **[Making API Requests](./05-making-api-requests.md)**
    Using `apiClient` and Repositories.
 
 6. **[Creating Pages](./06-creating-pages.md)**
    Server vs Client components.
 
 7. **[Creating APIs](./07-creating-apis.md)**
    Route handlers and security patterns.
 
 8. **[Using Hooks](./08-using-hooks.md)**
    Data fetching (Query) and modifying (Mutation).
 
 9. **[Adding a Feature](./09-adding-a-feature.md)**
    High-level checklist for comprehensive features.

 10. **[Server Actions](./10-server-actions.md)**
     Creating and using Server Actions for mutations.

 11. **[Feature Guards](./11-feature-guards.md)**
     Protecting features with auth, permission, and subscription guards.

 12. **[Subscription Patterns](./12-subscription-patterns.md)**
     Managing subscriptions, checkout, and billing with Polar.sh.

 ## Quick Reference
 
 - **Auth**: `getMemberSession()` (Server) / `useStytchMember()` (Client)
 - **Permissions**: `getServerPermissions()` (Server) / `usePermissions()` (Client)
 - **Data**: `apiClient.get()` or `useQuery()`
 - **Server Actions**: Server-side mutations with auth/permission guards
 
 ## Project Structure
 
 ```
 next_b2b_starter/
 ├── app/                    # Next.js App Router pages
 │   ├── api/               # API routes
 │   ├── auth/              # Login page
 │   └── dashboard/         # Protected pages
 ├── components/            # React components
 │   └── ui/               # shadcn/ui components
 ├── lib/                   # Utilities and business logic
 │   ├── api/              # API client and repositories
 │   ├── auth/             # Authentication utilities
 │   ├── hooks/            # React hooks
 │   └── contexts/         # React contexts
 ├── middleware.ts          # Route protection
 └── docs/                 # This documentation
 ```
 
 ## Core Concepts
 
 ### Server vs Client Components
 
 **Server Components** run on the server. Use them when you need:
 - Direct database/API access
 - Sensitive operations
 - SEO-optimized content
 
 **Client Components** run in the browser. Use them when you need:
 - Interactivity (buttons, forms)
 - Browser APIs
 - React hooks
 
 ### Repository Pattern
 
 Don't call APIs directly. Use repositories:
 
 ```typescript
 // Good
 const profile = await profileRepository.getProfile();
 
 // Avoid
 const response = await fetch('/api/profile');
 ```
 
 ### Permission-First Development
 
 Always check permissions before showing features:
 
 1. Define permission in `lib/auth/permissions.ts`
 2. Check permission before rendering UI
 3. Re-check permission in API route
 
 ## Need Help?
 
 Check `next_b2b_starter/README.md` for project-level info.
