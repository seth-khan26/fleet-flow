# Role-Based Access Control (RBAC)

FleetFlow uses a **flat role model** with a centralised permission map. Roles are stored on the `Membership` table and checked on every API request after the tenant context is established.

## Roles

| Role | Description |
|---|---|
| `OWNER` | Full access to everything; one per organisation |
| `ADMIN` | Same as OWNER minus `org.manage` |
| `DISPATCHER` | Manages the delivery queue; assigns drivers and vehicles |
| `FLEET_MANAGER` | Manages vehicles and maintenance; read-only on drivers |
| `DRIVER` | Can only read their own deliveries and update status |

A user can hold different roles in different organisations (modelled via the `Membership` join table).

## Permission Map

All role-to-permission assignments live in `src/lib/permissions.ts`. There are no inline role checks scattered across the codebase — everything goes through `hasPermission()`.

```typescript
const ROLE_PERMISSIONS: Record<MemberRole, Permission[]> = {
  OWNER: [
    'drivers.read', 'drivers.manage',
    'vehicles.read', 'vehicles.manage',
    'deliveries.read', 'deliveries.create', 'deliveries.dispatch', 'deliveries.update',
    'routes.read', 'routes.manage',
    'maintenance.read', 'maintenance.manage',
    'customers.read', 'customers.manage',
    'audit.read', 'users.manage', 'org.manage',
  ],
  DISPATCHER: [
    'deliveries.read', 'deliveries.create', 'deliveries.dispatch', 'deliveries.update',
    'drivers.read', 'vehicles.read',
    'routes.read', 'routes.manage',
    'customers.read',
  ],
  FLEET_MANAGER: [
    'vehicles.read', 'vehicles.manage',
    'maintenance.read', 'maintenance.manage',
    'drivers.read',
  ],
  DRIVER: [
    'deliveries.read', 'deliveries.update',
  ],
  // ADMIN inherits everything OWNER has except org.manage
}
```

## Permission Reference

| Permission | OWNER | ADMIN | DISPATCHER | FLEET_MANAGER | DRIVER |
|---|:---:|:---:|:---:|:---:|:---:|
| `deliveries.read` | ✓ | ✓ | ✓ | — | ✓ |
| `deliveries.create` | ✓ | ✓ | ✓ | — | — |
| `deliveries.dispatch` | ✓ | ✓ | ✓ | — | — |
| `deliveries.update` | ✓ | ✓ | ✓ | — | ✓ |
| `drivers.read` | ✓ | ✓ | ✓ | ✓ | — |
| `drivers.manage` | ✓ | ✓ | — | — | — |
| `vehicles.read` | ✓ | ✓ | ✓ | ✓ | — |
| `vehicles.manage` | ✓ | ✓ | — | ✓ | — |
| `maintenance.read` | ✓ | ✓ | — | ✓ | — |
| `maintenance.manage` | ✓ | ✓ | — | ✓ | — |
| `customers.read` | ✓ | ✓ | ✓ | — | — |
| `customers.manage` | ✓ | ✓ | — | — | — |
| `routes.read` | ✓ | ✓ | ✓ | — | — |
| `routes.manage` | ✓ | ✓ | ✓ | — | — |
| `audit.read` | ✓ | ✓ | — | — | — |
| `users.manage` | ✓ | ✓ | — | — | — |
| `org.manage` | ✓ | — | — | — | — |

## Enforcement Pattern

Every API route handler calls `requirePermission()` immediately after establishing the tenant context:

```typescript
export async function POST(req, { params }) {
  const org = await getOrganizationBySlug(params.orgSlug)
  const ctx = await getTenantContext(org.id)        // authenticate + verify membership
  await requirePermission(ctx, 'deliveries.create') // authorise

  // ... application logic
}
```

`requirePermission` throws `ForbiddenError` (HTTP 403) if the role does not include the requested permission. This propagates to `errorResponse()` at the top of the route handler, which serialises it and returns the response. No route handler can forget to check permissions — the function does not return until the check passes.

## Why Not RBAC Middleware

Putting the permission check in middleware would require the middleware to know which HTTP method + path combination maps to which permission. That coupling is fragile. Keeping the check inside the handler means the permission name is next to the logic it guards, making it easier to audit and harder to misconfigure.

## Design Decisions

**Flat roles, not hierarchical.** Roles are not a hierarchy (e.g. OWNER extends ADMIN). Each role has an explicit permission list. This prevents unintended permission inheritance if the hierarchy changes.

**Permissions are strings, not enums.** The `Permission` type is a union of string literals. This keeps the file self-contained and avoids a separate Prisma enum that would require a migration every time a new permission is added.

**No permission caching.** `hasPermission()` is a synchronous O(n) array scan over at most ~15 permissions. There is nothing to cache. The complexity does not justify it.

## Testing

`src/__tests__/permissions.test.ts` covers:
- OWNER has all permissions
- DISPATCHER can dispatch but cannot manage drivers or vehicles
- DRIVER can only read and update deliveries
- FLEET_MANAGER can manage vehicles and maintenance but cannot dispatch
- OWNER has at least as many permissions as every other role

## Dispatcher View

Real-time view of active deliveries and driver locations.

## Utilization Analytics

Vehicle idle time, active hours, and utilization rate per fleet.
