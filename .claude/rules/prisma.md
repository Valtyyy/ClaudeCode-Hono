# Prisma — Mandatory Rules

## Singleton Client

`PrismaClient` is instantiated exactly once, in `src/lib/prisma.ts`, and exported as `prisma`. Every other file imports from there. Never call `new PrismaClient()` anywhere else.

```typescript
// src/lib/prisma.ts
import { PrismaClient } from '@prisma/client'
const globalForPrisma = globalThis as unknown as { prisma: PrismaClient }
export const prisma =
  globalForPrisma.prisma ?? new PrismaClient({ log: [...] })
if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = prisma
```

## Schema Conventions

- All models use `@id @default(autoincrement())` integer primary keys — **except** the Better Auth-managed models (`User`, `Session`, `Account`, `Verification`), which use Better Auth's own string ID scheme. Never change their ID type to match app conventions.
- All models include `createdAt DateTime @default(now())`. Mutable models also include `updatedAt DateTime @updatedAt`.
- Relation ownership sits on the side that holds the foreign key. Back-reference fields on the other side are added only when Prisma requires them for the relation to be valid — they are not exposed in API responses unless explicitly needed.
- Enums are defined in the schema and used for any finite set of string values (providers, statuses, …). Roles are the one exception — they are owned by Better Auth's `admin` plugin as a string field, not a schema enum (see `auth.md`).
- Many-to-many relations use explicit named `@relation("Name")` when two relations exist between the same pair of models.
- One-to-many is preferred over many-to-many unless the domain genuinely requires shared ownership of the child record.

## Better Auth-Managed Models

`User`, `Session`, `Account`, and `Verification` in `prisma/schema.prisma` are owned by Better Auth, not hand-written:

- Generate/update them with `npx @better-auth/cli generate --output prisma/schema.prisma` whenever `src/lib/auth.ts` changes (new plugin, new field, …), then run the normal migration commands below.
- Never hand-edit these four models directly in `schema.prisma` — edit the Better Auth config instead and regenerate. Manual edits get silently overwritten or drift from what `auth.api.*` expects at runtime.
- App-owned models may still hold a foreign key to `User.id` (e.g. `authorId String` referencing the Better Auth user) — that relation is normal Prisma and is not affected by the rule above.

## Pagination — Never Use Bare `findMany()`

Never call `prisma.<model>.findMany()` without `take` and `skip`. Every list query must be paginated with a hard cap of 100 items per page.

```typescript
// FORBIDDEN — unbounded query
const items = await prisma.item.findMany({ where: { ... } })

// CORRECT — paginated, cap enforced in the Zod schema
const ListQuerySchema = z.object({
  page:     z.coerce.number().int().min(1).default(1),
  pageSize: z.coerce.number().int().min(1).max(100).default(20),
})

const listItemsRoute = createRoute({
  // ...
  request: { query: ListQuerySchema },
  responses: {
    200: { content: { 'application/json': { schema: ListResponseSchema } }, description: 'OK' },
  },
})

// handler:
itemsRouter.openapi(listItemsRoute, async (c) => {
  const { page, pageSize } = c.req.valid('query')
  const [items, total] = await Promise.all([
    prisma.item.findMany({ skip: (page - 1) * pageSize, take: pageSize }),
    prisma.item.count(),
  ])
  return c.json({ items, total, page, pageSize }, 200)
})
```

Rules:
- `page` and `pageSize` must be declared in a named query schema using `z.coerce.number()` — query strings are always coerced, never validated as raw strings.
- Cap `pageSize` with `.max(100)` in the Zod schema — never pass a `take` value greater than 100.
- Always access via `c.req.valid('query')` — never call `c.req.query()` directly.
- Include `total` in the response body when the consumer needs it (`prisma.<model>.count({ where })`).

## Migrations

After any app-model schema change run:
```bash
npx prisma migrate dev --name <description>
npx prisma generate
```

After any Better Auth config change (new plugin, new field, …), first regenerate the Better Auth-managed models, then run the same two commands:
```bash
npx @better-auth/cli generate --output prisma/schema.prisma
npx prisma migrate dev --name <description>
npx prisma generate
```

Never edit the database directly. Never commit without running `generate`.
