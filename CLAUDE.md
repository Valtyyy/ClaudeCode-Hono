# CLAUDE.md — Hono + Prisma API

This file defines the rules, conventions, and architecture Claude must follow at all times when working on this codebase. These rules are non-negotiable and apply to every file created or modified, regardless of the scope of the change.

---

## Stack

| Layer       | Technology                              |
|-------------|----------------------------------------|
| Runtime     | Node.js via `@hono/node-server`        |
| Framework   | Hono (`OpenAPIHono` from `@hono/zod-openapi`) |
| ORM         | Prisma                                 |
| Validation  | Zod via `createRoute()` (`@hono/zod-openapi`) |
| OpenAPI     | `@hono/zod-openapi` + `@hono/swagger-ui` |
| Auth        | Better Auth (`better-auth`) + Prisma adapter |
| Testing     | Vitest                                 |
| Language    | TypeScript (strict mode)               |

---

## Project Structure

```
src/
├── index.ts           # Entry point: global middleware + app.route() mounts + the Better Auth catch-all handler
├── lib/
│   ├── prisma.ts      # Prisma singleton — the only place PrismaClient is instantiated
│   └── auth.ts        # Better Auth singleton — the only place betterAuth() is instantiated
├── middleware/        # Reusable middleware, one concern per file
├── routes/            # One Hono sub-app per resource
├── types.ts           # Shared TypeScript types (AppVariables, etc.)
└── tests/
    ├── setup.ts       # Global Vitest setup: vi.mock Prisma + vi.mock Better Auth + env vars
    ├── helpers.ts     # Shared test utilities (makeSession, makeUser, …)
    ├── middleware/    # Unit tests for middleware in isolation
    └── routes/        # Integration tests per route file
prisma/
└── schema.prisma
```

New resources follow the same pattern: one file in `src/routes/`, mounted in `src/index.ts` via `app.route()`, tested in `src/tests/routes/`. No exceptions — except the Better Auth handler itself, which is a raw catch-all mount (see `auth.md`), not a resource route.

---

## File Reading Rules

- Never read more than 3 files per task unless explicitly told to.
- If the task concerns a single file, read ONLY that file and its direct imports.
- Never explore the codebase to "understand the project" — CLAUDE.md exists for that.

---

## Adding a New Resource — Checklist

Follow this checklist exactly when adding a new model, routes, or middleware.

**Prisma**
- [ ] Add the model to `prisma/schema.prisma` following schema conventions (never hand-edit the Better Auth-managed models — see `prisma.md`)
- [ ] Run `npx prisma migrate dev --name <description>` and `npx prisma generate`
- [ ] Add the new Prisma model methods to the mock in `src/tests/setup.ts`

**Route file**
- [ ] Create `src/routes/<resource>.ts` exporting a `new OpenAPIHono<{ Variables: AppVariables }>()` instance
- [ ] Declare Zod schemas as named variables at the top, then `createRoute()` declarations, then `.openapi()` calls with inline handlers
- [ ] Each `.openapi()` call must be preceded by a `// [METHOD] /path` comment
- [ ] Use `c.req.valid()` for all input; use `HTTPException` for all errors
- [ ] Apply `authMiddleware` and `requireRoles(...)` via `router.use('/*', ...)` or per-route; leave public routes without any middleware

**Mounting**
- [ ] Import the new route file in `src/index.ts` and mount with `app.route('/path', resource)`

**Types**
- [ ] If new context variables are needed, add them to `AppVariables` in `src/types.ts`

**Tests**
- [ ] Create `src/tests/routes/<resource>.test.ts`
- [ ] Cover: happy path, 401, 403, 400, 404 for each route
- [ ] If a new middleware is added: create `src/tests/middleware/<middleware>.test.ts` and test it in isolation

**New middleware**
- [ ] Define using `createMiddleware<{ Variables: AppVariables }>()` from `hono/factory`
- [ ] Export from `src/middleware/<name>.ts`
- [ ] Test in isolation in `src/tests/middleware/<name>.test.ts`

---

## What Claude Must Never Do

- Instantiate `PrismaClient` outside `src/lib/prisma.ts`
- Instantiate `betterAuth()` outside `src/lib/auth.ts`
- Sign, verify, or decode a session/token by hand — always go through `auth.api.*`
- Use `app.get()` / `app.post()` for API routes — all routes must use `createRoute()` + `app.openapi()` (the `/api/auth/*` catch-all in `auth.md` is the sole exception)
- Extract a route handler into a named function and pass it by reference — handlers are always inline anonymous functions inside `.openapi()`
- Define anonymous schemas inline inside `createRoute()` — schemas must be declared as named variables above
- Split a resource into multiple route files unless a schema is explicitly imported by another resource
- Write a reusable middleware without `createMiddleware()`
- Validate request input manually inside a handler — use `c.req.valid()` from the `createRoute()` schema
- Call `c.req.json()`, `c.req.param()`, or `c.req.query()` directly in handlers
- Throw raw errors or return `c.json({ error })` for HTTP errors — use `HTTPException`
- Define a `app.head()` route
- Import route files inside middleware test files
- Use `authMiddleware` inside a `requireRoles` test
- Let Vitest tests connect to a real database
- Duplicate Prisma mock definitions across test files instead of centralizing in `setup.ts`
- Use `any` without a justifying comment
- Skip writing tests when adding a new route or middleware file

---

@.claude/rules/openapi.md
@.claude/rules/hono.md
@.claude/rules/prisma.md
@.claude/rules/auth.md
@.claude/rules/testing.md
@.claude/rules/typescript.md
