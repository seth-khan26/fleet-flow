# Architecture Overview

FleetFlow is built as a **modular monolith** — all code lives in one Next.js application, but is organised into clearly bounded modules that could be extracted into services if the product ever scales past the point where a monolith makes sense.

## Why a Modular Monolith

Microservices add operational overhead (service discovery, distributed tracing, inter-service auth, network latency on every call) that a new product cannot justify. A modular monolith lets us:

- Share a single Prisma client and database connection pool
- Keep transactions atomic across multiple domain concerns
- Deploy as a single unit (one Docker image, one health-check endpoint)
- Extract modules later without rewriting the domain logic — the boundaries are already there

## Stack

| Layer | Choice | Reason |
|---|---|---|
| Framework | Next.js 14 (App Router) | RSC for server-side data fetching, API Routes for the REST surface, single deploy target |
| ORM | Prisma 7 + `@prisma/adapter-pg` | Type-safe query builder, migration tooling, raw SQL escape hatch for locking |
| Database | PostgreSQL | ACID transactions, row-level locking (`SELECT FOR UPDATE`), RLS for defence-in-depth |
| Auth | NextAuth.js v5 (beta) | JWT sessions, Credentials provider, PrismaAdapter wires the session store |
| UI | shadcn/ui + Tailwind CSS | Accessible Radix primitives, utility-first styling, no runtime CSS-in-JS |
| Validation | Zod | Schema-first validation shared between API input parsing and form resolvers |
| Forms | React Hook Form + `@hookform/resolvers/zod` | Uncontrolled inputs, minimal re-renders, Zod schema reuse |
| Notifications | `setImmediate` fire-and-forget | Avoids adding Redis for MVP; swappable for BullMQ later without touching callers |

## Request Flow

Every API route follows the same layered pipeline. Nothing skips a layer.

```
HTTP Request
    │
    ▼
Next.js Middleware (src/middleware.ts)
    │  JWT verification via NextAuth
    │  Redirect unauthenticated requests to /login
    │  Allow /api/auth/* and /api/onboarding unconditionally
    │
    ▼
Route Handler (src/app/api/[orgSlug]/...)
    │
    ▼
getOrganizationBySlug(slug)
    │  Resolves the URL slug to an org ID
    │  Throws NotFoundError (404) if unknown slug
    │
    ▼
getTenantContext(org.id)          ← src/lib/tenant.ts
    │  Reads JWT session → user ID
    │  Queries Membership table for (userId, organizationId)
    │  Throws ForbiddenError (403) if not a member
    │  Returns { userId, organizationId, role }
    │
    ▼
requirePermission(ctx, 'resource.action')
    │  Central permission check — no inline role comparisons elsewhere
    │  Throws ForbiddenError (403) on failure
    │
    ▼
Zod input validation
    │  schema.parse(body)
    │  Throws ZodError → serialised as 400 Bad Request
    │
    ▼
Application / Domain Logic
    │  Business rules, state machine checks, availability checks
    │
    ▼
db.$transaction(async (tx) => { ... })
    │  All mutations run inside a transaction
    │  Audit log written atomically in the same transaction
    │
    ▼
queueNotification(...)            ← post-commit, fire-and-forget
    │  setImmediate ensures the HTTP response is sent first
    │
    ▼
Response.json(result, { status: 2xx })
```

If anything throws an `AppError` subclass at any point, `errorResponse()` converts it to the correct HTTP status code. Unhandled errors produce a generic 500.

## Module Structure

```
src/
├── app/
│   ├── (dashboard)/[orgSlug]/     # Admin/dispatcher UI pages (RSC)
│   │   ├── page.tsx               # Dashboard stats
│   │   ├── deliveries/            # List + detail pages
│   │   ├── dispatch/              # Kanban board
│   │   ├── drivers/               # Driver management
│   │   ├── vehicles/              # Vehicle fleet
│   │   ├── customers/             # Customer management
│   │   ├── routes/                # Route planner
│   │   ├── maintenance/           # Maintenance records
│   │   ├── audit-logs/            # Audit trail (admin only)
│   │   └── settings/              # Org settings + team
│   ├── driver/[orgSlug]/          # Driver mobile interface
│   ├── api/[orgSlug]/             # REST API handlers
│   ├── login/                     # Auth pages
│   └── register/
├── components/
│   ├── ui/                        # shadcn/ui primitives
│   ├── layout/                    # Sidebar, Header
│   ├── deliveries/                # Delivery-specific components
│   ├── dispatch/                  # Kanban board components
│   ├── driver/                    # Driver mobile components
│   └── shared/                    # Reusable dialogs (create forms)
├── lib/
│   ├── auth.ts                    # NextAuth configuration
│   ├── db.ts                      # Prisma singleton
│   ├── tenant.ts                  # Tenant context + permission guard
│   ├── permissions.ts             # RBAC permission map
│   ├── delivery-state-machine.ts  # Allowed status transitions
│   ├── audit.ts                   # Audit log writer
│   ├── notifications.ts           # Async notification queue
│   ├── errors.ts                  # Typed error hierarchy
│   └── utils.ts                   # Formatting helpers, cn()
├── types/
│   └── index.ts                   # Re-exports Prisma types + composite types
└── __tests__/
    ├── delivery-state-machine.test.ts
    └── permissions.test.ts
```
