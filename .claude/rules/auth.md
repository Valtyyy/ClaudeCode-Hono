# Authentication & Authorization

## Auth Flow

1. `authMiddleware` (`src/middleware/auth.ts`) reads `Authorization: Bearer <token>`, verifies the JWT with `hono/jwt`, loads the user with their roles from Prisma, and stores the result in `c.var.user`.
2. `requireRoles(...roles)` (`src/middleware/roles.ts`) reads `c.var.user` and checks role membership. The `admin` role always passes, checked first.
3. Routes that need protection chain `authMiddleware` then `requireRoles(...)` before the handler.
4. Routes without any middleware are public by default.

## Middleware Order

```typescript
// Route-level protection (mixed public/protected in the same file)
app.post('/', authMiddleware, requireRoles('bot'), zValidator('json', schema), async (c) => { ... })

// File-level protection (all routes in the file are protected)
app.use('/*', authMiddleware, requireRoles('bot'))
```

`requireRoles` must never appear before `authMiddleware`. It reads `c.var.user` which `authMiddleware` sets.

## Adding a New Protected Route

1. Decide the required role(s) based on the permissions table in the project spec.
2. Apply `authMiddleware` then `requireRoles('role')` to that specific route, or to the whole file via `app.use('/*', ...)` if all routes share the same protection.
3. Public routes need no middleware at all — do not add unnecessary middleware.
