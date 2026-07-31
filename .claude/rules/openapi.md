# OpenAPI — Mandatory Rules

All API routes in this project use `@hono/zod-openapi`. The rules below are non-negotiable and apply to every route file created or modified.

---

## Hard Constraints

- **Every Hono route MUST use `createRoute()` from `@hono/zod-openapi` and be registered via `app.openapi()`. Direct `app.get()` / `app.post()` for API routes is forbidden.**
- **Splitting a resource into multiple files is forbidden unless a schema is explicitly imported by another resource.**
- **Every `.openapi()` call must be preceded by a `// [METHOD] /path` comment.**
- **Handlers are inline anonymous functions inside `.openapi()` — never extracted into named functions.**

---

## Router Type

Route files use `OpenAPIHono`, not plain `Hono`:

```typescript
import { OpenAPIHono, createRoute } from '@hono/zod-openapi'
import type { AppVariables } from '../types'

export const resourceRouter = new OpenAPIHono<{ Variables: AppVariables }>()
```

`src/index.ts` assembles the root app from `OpenAPIHono` and registers the spec and UI endpoints:

```typescript
import { OpenAPIHono } from '@hono/zod-openapi'
import { swaggerUI } from '@hono/swagger-ui'

const app = new OpenAPIHono()

app.route('/users', userRouter)
// …other resource routers

app.doc('/doc', { openapi: '3.0.0', info: { title: 'API', version: '1.0.0' } })
app.get('/ui', swaggerUI({ url: '/doc' }))
```

---

## Internal File Order

Within every `src/routes/<resource>.ts` file, sections must appear in this exact order:

1. **Zod schemas** — named variables, exported only if another resource imports them
2. **`createRoute()` declarations** — one `const {action}Route = createRoute({ … })` per route
3. **Router assembly** — `export const {resource}Router` with `.openapi()` calls, each preceded by `// [METHOD] /path`

---

## Schemas

Declare every schema as a named variable above `createRoute()`. Never define a schema inline inside `createRoute()`.

```typescript
// FORBIDDEN
const getItemRoute = createRoute({
  responses: {
    200: { content: { 'application/json': { schema: z.object({ id: z.string() }) } }, description: 'OK' },
  },
})

// CORRECT
const ItemSchema = z.object({ id: z.string(), name: z.string() })

const getItemRoute = createRoute({
  responses: {
    200: { content: { 'application/json': { schema: ItemSchema } }, description: 'OK' },
  },
})
```

---

## `createRoute()` Declaration Shape

```typescript
const {action}Route = createRoute({
  method: 'get' | 'post' | 'put' | 'patch' | 'delete',
  path: '/resource/{id}',          // OpenAPI path template — {param}, not :param
  summary: 'Short description',
  tags: ['ResourceName'],
  request: {
    params: z.object({ id: z.string() }),             // path params
    query:  z.object({ page: z.string().optional() }), // query string
    body: { content: { 'application/json': { schema: BodySchema } } },
  },
  responses: {
    200: { content: { 'application/json': { schema: ResponseSchema } }, description: 'OK' },
    400: { description: 'Bad request' },
    401: { description: 'Unauthorized' },
    404: { description: 'Not found' },
  },
})
```

Declare only the responses the handler can actually return.

---

## Router Assembly & Handlers

Handlers are inline anonymous functions inside `.openapi()`. Each call is preceded by `// [METHOD] /path`. Use `c.req.valid()` to access validated input — never `c.req.json()` or `c.req.param()` directly.

```typescript
export const userRouter = new OpenAPIHono<{ Variables: AppVariables }>()

// [GET] /users/{id}
userRouter.openapi(getUserRoute, async (c) => {
  const { id } = c.req.valid('param')
  const user = await prisma.user.findUnique({ where: { id: Number(id) } })
  if (!user) throw new HTTPException(404, { message: 'Not found' })
  return c.json(user, 200)
})

// [POST] /users
userRouter.openapi(createUserRoute, async (c) => {
  const body = c.req.valid('json')
  const user = await prisma.user.create({ data: body })
  return c.json(user, 201)
})
```

Auth middleware is applied via `router.use('/*', authMiddleware, requireRoles('role'))` before the `.openapi()` calls, or per-route by passing middleware in the `createRoute()` `middleware` array.

---

## Canonical Example

```typescript
// src/routes/users.ts
import { OpenAPIHono, createRoute } from '@hono/zod-openapi'
import { HTTPException } from 'hono/http-exception'
import { z } from 'zod'
import { prisma } from '../lib/prisma'
import { authMiddleware } from '../middleware/auth'
import { requireRoles } from '../middleware/roles'
import type { AppVariables } from '../types'

// ── Schemas ──────────────────────────────────────────────────────────────────

const UserSchema = z.object({ id: z.number(), name: z.string() })
const CreateUserSchema = z.object({ name: z.string().min(1) })

// ── Routes ───────────────────────────────────────────────────────────────────

const getUserRoute = createRoute({
  method: 'get',
  path: '/users/{id}',
  summary: 'Get a user',
  tags: ['Users'],
  request: { params: z.object({ id: z.string() }) },
  responses: {
    200: { content: { 'application/json': { schema: UserSchema } }, description: 'OK' },
    404: { description: 'Not found' },
  },
})

const createUserRoute = createRoute({
  method: 'post',
  path: '/users',
  summary: 'Create a user',
  tags: ['Users'],
  request: { body: { content: { 'application/json': { schema: CreateUserSchema } } } },
  responses: {
    201: { content: { 'application/json': { schema: UserSchema } }, description: 'Created' },
  },
})

// ── Router ───────────────────────────────────────────────────────────────────

export const userRouter = new OpenAPIHono<{ Variables: AppVariables }>()

// [GET] /users/{id}
userRouter.openapi(getUserRoute, async (c) => {
  const { id } = c.req.valid('param')
  const user = await prisma.user.findUnique({ where: { id: Number(id) } })
  if (!user) throw new HTTPException(404, { message: 'Not found' })
  return c.json(user, 200)
})

// [POST] /users
userRouter.openapi(createUserRoute, async (c) => {
  const body = c.req.valid('json')
  const user = await prisma.user.create({ data: body })
  return c.json(user, 201)
})
```
