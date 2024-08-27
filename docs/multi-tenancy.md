# Multi-Tenancy

FleetFlow uses a **shared-database, shared-schema** multi-tenancy model. Every tenant's data lives in the same tables, separated by an `organizationId` column that is present on every tenant-owned table.

## Data Model

```
User ─────────────── Membership ─────────────── Organization
(global identity)    (role per org)              (tenant root)
                          │
                          └── role: OWNER | ADMIN | DISPATCHER
                                    FLEET_MANAGER | DRIVER

Organization
  ├── Driver[]
  ├── Vehicle[]
  ├── Customer[]
  ├── Delivery[]
  ├── Route[]
  ├── MaintenanceRecord[]
  ├── AuditLog[]
  └── Notification[]
```

A `User` is global — the same account can be a member of multiple organizations with different roles in each. The `Membership` join table holds the role per (user, org) pair.

## Tenant Context Enforcement

The pattern that enforces tenant isolation is `getTenantContext()` in `src/lib/tenant.ts`. It is called at the top of every API route handler:

```typescript
// 1. Resolve slug → org ID (never trust the client to supply an org ID)
const org = await getOrganizationBySlug(orgSlug)

// 2. Verify the authenticated user is a member of that org
const ctx = await getTenantContext(org.id)
// → throws ForbiddenError (403) if the user has no Membership record

// 3. Check the user has the required permission for this action
await requirePermission(ctx, 'deliveries.dispatch')
// → throws ForbiddenError (403) if their role lacks the permission
```

The `organizationId` used in all subsequent queries comes from `ctx.organizationId` — the value that was validated against the database. It is never taken from the request body or query parameters. This prevents a malicious client from supplying a foreign tenant's ID to access another tenant's data.

## Prisma Query Pattern

Every query that reads or writes tenant data includes `organizationId: ctx.organizationId` in its `where` clause:

```typescript
// Always scoped — never query without organizationId
const delivery = await db.delivery.findFirst({
  where: { id, organizationId: ctx.organizationId },
})
// If the delivery belongs to another tenant, findFirst returns null
// → handled as NotFoundError (404), not a 403
// This avoids leaking the existence of resources in other tenants
```

Returning 404 (rather than 403) when a cross-tenant ID is supplied is intentional: it does not reveal whether the resource exists at all.

## PostgreSQL Row-Level Security

The schema is designed to support PostgreSQL RLS as a defence-in-depth layer. In production you can enable RLS policies on all tenant tables that verify `current_setting('app.tenant_id') = "organizationId"`. The application sets this session variable before executing queries:

```sql
-- Example RLS policy (production hardening)
ALTER TABLE "Delivery" ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON "Delivery"
  USING ("organizationId" = current_setting('app.tenant_id'));
```

Even if a bug in application code omitted the `organizationId` filter, the database itself would refuse to return rows belonging to the wrong tenant. The application-level checks are the primary gate; RLS is the safety net.

## Organization Slug in URLs

Routes are namespaced by organization slug (`/{orgSlug}/...`). The slug is a URL-safe string (lowercase alphanumeric + hyphens) set at registration time. It serves as the tenant's stable URL namespace without exposing internal database IDs in the URL.

The slug is always resolved to an `organizationId` through a database lookup before any tenant context is established. Client-supplied slugs are never trusted directly.

## Cross-Tenant Security Tests

The test suite includes mandatory cross-tenant isolation checks:

- Tenant A cannot read Tenant B's deliveries, drivers, vehicles, customers, or maintenance records
- Supplying a valid resource ID that belongs to another tenant returns 404, not the resource
- `getTenantContext()` rejects any user not listed in the org's Membership table
