# Testing — Mandatory Rules

All tests use **Vitest**. There is no test database. Prisma is mocked globally.

## What Must Be Tested

Every file in `src/middleware/` and `src/routes/` must have a corresponding test file. No new middleware or route file is considered complete without tests.

| File added                      | Test file required                        |
|--------------------------------|------------------------------------------|
| `src/middleware/foo.ts`        | `src/tests/middleware/foo.test.ts`       |
| `src/routes/bar.ts`            | `src/tests/routes/bar.test.ts`           |

## Middleware Tests — Isolation Rules

Middleware is tested by mounting it on a **throwaway `new Hono()` app** created inside the test file. The actual route files are never imported in middleware tests.

For `authMiddleware`: test missing/absent session (`auth.api.getSession` mock resolves `null`) and a valid session (`auth.api.getSession` mock resolves `{ user, session }`).

For `requireRoles`: inject the user directly into `c.var` via a preceding inline middleware — do not involve `authMiddleware`. Test: matching role, one of multiple roles, `admin` bypass, no matching role, wrong role.

```typescript
// Injecting a user without authMiddleware
const withUser = (user: AppVariables['user']) => async (c: any, next: any) => {
  c.set('user', user)
  await next()
}
```

## Route Tests — What to Cover

For every route:
- **Happy path**: correct request, correct mock return, expected status and body.
- **Auth guard**: 401 when `Authorization` header is absent (for every protected route).
- **Role guard**: 403 when the user has no matching role (for every role-protected route).
- **Validation**: 400 when the request body or params fail the Zod schema.
- **Not found**: 404 when the Prisma mock returns `null` or throws.

OAuth routes specifically: test both the **new user** path (account not found → user created) and the **existing user** path (account found → user returned) for **every supported provider** individually.

## Prisma Mock

Prisma is mocked in `src/tests/setup.ts` via `vi.mock('../lib/prisma', ...)`. Every method used by the codebase (`findUnique`, `findMany`, `create`, `update`, `delete`) is declared as a `vi.fn()` there.

Call `vi.clearAllMocks()` in `beforeEach` in every test file. Never let mock state leak between tests.

## Better Auth Mock

Better Auth is mocked in `src/tests/setup.ts` via `vi.mock('../lib/auth', ...)`, with `auth.api.getSession` declared as a `vi.fn()`. Route and middleware tests never hit a real Better Auth instance or a real database through the Prisma adapter — they only control what `getSession` resolves to.

When a route handler is protected by `authMiddleware`, always set `auth.api.getSession` to resolve the desired session (or `null`) before firing the request — even in route tests that are not about auth.

## Session Generation in Tests

Do not sign or construct a real Better Auth session token. Use `makeSession(user)` (see below) to build the `{ user, session }` shape Better Auth returns, and set it as the resolved value of the mocked `auth.api.getSession`:

```typescript
vi.mocked(auth.api.getSession).mockResolvedValueOnce(makeSession(makeAdminUser()))
```

## Test Helpers

`src/tests/helpers.ts` must export at minimum:
- `makeSession(user)` — returns the `{ user, session }` shape returned by `auth.api.getSession`
- `makeUser(overrides?)` — base user object, including the Better Auth `role` field
- `makeBotUser()`, `makeAdminUser()`, `makeNoRoleUser()` — pre-configured users

Add helpers to this file whenever a new user shape or fixture is needed across multiple test files. Never duplicate fixture definitions across test files.
