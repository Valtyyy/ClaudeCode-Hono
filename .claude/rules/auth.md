# Authentication & Authorization

Authentication is handled entirely by **Better Auth** (`better-auth`), backed by the Prisma adapter against the shared `prisma` singleton. Nothing in this codebase signs, verifies, or decodes a token by hand.

## Better Auth Instance

`betterAuth()` is instantiated exactly once, in `src/lib/auth.ts`, and exported as `auth` — mirroring the Prisma singleton rule in `prisma.md`. Never call `betterAuth()` anywhere else.

```typescript
// src/lib/auth.ts
import { betterAuth } from 'better-auth'
import { prismaAdapter } from 'better-auth/adapters/prisma'
import { bearer } from 'better-auth/plugins/bearer'
import { admin } from 'better-auth/plugins/admin'
import { prisma } from './prisma'

export const auth = betterAuth({
  database: prismaAdapter(prisma, { provider: 'postgresql' }),
  emailAndPassword: { enabled: true },
  plugins: [bearer(), admin()],
})
```

The `bearer` plugin is required — this is an API-only backend with no server-rendered pages, so clients authenticate with `Authorization: Bearer <session-token>` rather than relying on cookies.

The `admin` plugin is required for roles — see below. Never model roles with a hand-rolled `user.additionalFields.role` or a custom Prisma relation; Better Auth already owns this via the plugin.

## Mounting the Auth Handler

Better Auth's own handler is mounted as a catch-all in `src/index.ts`:

```typescript
app.on(['POST', 'GET'], '/api/auth/*', (c) => auth.handler(c.req.raw))
```

**This is the one and only exception to the `openapi.md` rule that every route must use `createRoute()` + `app.openapi()`.** Better Auth owns the request/response contract for `/api/auth/*` — sign-in, sign-up, session, OAuth callbacks, etc. — and validates it internally. Never hand-write `createRoute()` declarations, Zod schemas, or `.openapi()` calls for paths under `/api/auth/*`, and never re-implement any of that logic in a route file.

## Auth Flow

1. `authMiddleware` (`src/middleware/auth.ts`) calls `auth.api.getSession({ headers: c.req.raw.headers })`. If it resolves to `null`, throw `HTTPException(401)`. Otherwise store `session.user` in `c.var.user`.
2. `requireRoles(...roles)` (`src/middleware/roles.ts`) reads `c.var.user.role` and checks membership. `admin` always passes, checked first.
3. Routes that need protection chain `authMiddleware` then `requireRoles(...)` before the handler.
4. Routes without any middleware are public by default.

## Roles

Roles are entirely owned by Better Auth's `admin` plugin (`better-auth/plugins/admin`), enabled in `src/lib/auth.ts` (see above) — never hand-roll role storage.

- The plugin adds a `role` field (string, default `'user'`) plus ban-related fields to the Better Auth-managed `user` model. Do not add a competing `role` field via `user.additionalFields`, and do not model roles as a separate Prisma relation table — both would fight the plugin's own schema and migrations.
- Multiple roles per user are represented as a comma-separated string on that same field (e.g. `'user,editor'`), per the admin plugin's own convention. `requireRoles` reads `c.var.user.role` (present because `authMiddleware` stores `session.user`, which the plugin extends) and must split on `,` when checking membership — never assume a single value.
- Re-run `npx @better-auth/cli generate --output prisma/schema.prisma` after enabling or reconfiguring the `admin` plugin — it changes the generated `User` model — then `npx prisma migrate dev`.
- Server-side role/permission checks that go beyond simple membership (e.g. fine-grained permissions) should use `auth.api.userHasPermission(...)` rather than reimplementing access-control logic — see the Better Auth admin plugin docs.

## Middleware Order

```typescript
// Route-level protection (mixed public/protected in the same file)
router.openapi(createBotResourceRoute, /* preceded by authMiddleware + requireRoles applied to this route */ async (c) => { ... })

// File-level protection (all routes in the file are protected)
router.use('/*', authMiddleware, requireRoles('bot'))
```

`requireRoles` must never appear before `authMiddleware`. It reads `c.var.user`, which `authMiddleware` sets.

## Adding a New Protected Route

1. Decide the required role(s) based on the permissions table in the project spec.
2. Apply `authMiddleware` then `requireRoles('role')` to that specific route, or to the whole file via `router.use('/*', ...)` if all routes share the same protection.
3. Public routes need no middleware at all — do not add unnecessary middleware.

## What Changed From Custom JWT Auth

- No more `hono/jwt`, no manual `sign()` / `verify()` calls, no `JWT_SECRET`. See `typescript.md` for the replacement env vars (`BETTER_AUTH_SECRET`, `BETTER_AUTH_URL`).
- Session lookups always go through `auth.api.getSession(...)` — never decode or inspect a token by hand in a handler or middleware.
- Better Auth manages its own Prisma models (`User`, `Session`, `Account`, `Verification`); see the Better Auth section in `prisma.md` for the schema and migration implications.
