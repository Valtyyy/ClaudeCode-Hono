# Hono — Mandatory Rules

These rules come directly from Hono's official best practices and must never be violated.

## No Extracted Handler Functions

Never extract a handler into a named function and pass it by reference. Handlers are always inline anonymous functions — either inside `app.openapi()` (the standard for API routes) or inside `app.get()` / `app.post()` if those are ever used for non-API endpoints. Extracting a handler breaks type inference for path parameters and context variables.

```typescript
// FORBIDDEN — extracted named function passed by reference
const listItems = (c: Context) => { ... }
app.get('/items', listItems)

// FORBIDDEN — same problem with openapi
function handleListItems(c) { ... }
app.openapi(listItemsRoute, handleListItems)

// CORRECT — handler inline inside .openapi()
app.openapi(listItemsRoute, async (c) => { ... })
```

See `openapi.md` for the full `.openapi()` pattern, which is the required form for all API routes.

## Route Modules via `app.route()`

Each resource lives in its own file exporting an `OpenAPIHono` instance. `src/index.ts` only mounts them — it contains no business logic.

```typescript
// src/routes/items.ts
import { OpenAPIHono } from '@hono/zod-openapi'

export const itemsRouter = new OpenAPIHono<{ Variables: AppVariables }>()
itemsRouter.openapi(listItemsRoute, async (c) => { ... })

// src/index.ts
import { itemsRouter } from './routes/items'
app.route('/items', itemsRouter)
```

## `createMiddleware()` for All Reusable Middleware

Any middleware that is defined outside an inline `app.use()` call must use `createMiddleware()` from `hono/factory`. This preserves type inference for `c.var` across the chain.

```typescript
import { createMiddleware } from 'hono/factory'

export const myMiddleware = createMiddleware<{ Variables: AppVariables }>(
  async (c, next) => {
    // ...
    await next()
  }
)
```

Never write a bare `async (c, next) => {}` function and export it as middleware.

## Typed Context Variables

Declare a shared `AppVariables` type in `src/types.ts` and pass it as the Hono generic on every sub-app and middleware that reads from `c.var`.

```typescript
// src/types.ts
export type AppVariables = {
  user: User & { roles: { name: string }[] }
  // add new variables here as the app grows
}

// src/routes/anything.ts
const app = new Hono<{ Variables: AppVariables }>()
```

Never use `c.set` / `c.get` / `c.var` without the correct generic in scope.

## Validation — Never Manual

All request input is validated via the route declaration — never manually inside a handler.

For API routes (all routes), validation is declared in `createRoute()` request schemas. The handler accesses parsed values via `c.req.valid('param' | 'json' | 'query')`:

```typescript
// createRoute() declares the schema
const createItemRoute = createRoute({
  request: { body: { content: { 'application/json': { schema: CreateItemSchema } } } },
  // ...
})

// Handler receives already-validated, typed input
app.openapi(createItemRoute, async (c) => {
  const body = c.req.valid('json') // typed, already validated
  // ...
})
```

Never call `c.req.json()`, `c.req.param()`, or `c.req.query()` directly, and never validate input manually inside a handler. `zValidator` from `@hono/zod-validator` is not used — `createRoute()` replaces it.

## Error Handling via `HTTPException`

All HTTP errors are thrown as `HTTPException` from `hono/http-exception`. A global `app.onError` handler in `src/index.ts` catches them and formats the response. Never return raw error responses from handlers or middleware.

```typescript
import { HTTPException } from 'hono/http-exception'

// Inside a handler or middleware:
throw new HTTPException(404, { message: 'Not found' })
throw new HTTPException(403, { message: 'Forbidden' })

// src/index.ts — the single place that formats errors:
app.onError((err, c) => {
  if (err instanceof HTTPException) return c.json({ error: err.message }, err.status)
  console.error(err)
  return c.json({ error: 'Internal Server Error' }, 500)
})
```

## HEAD Requests

Never define dedicated `app.head()` routes. Hono automatically handles HEAD by converting it to GET and stripping the body. Rely on this behavior.
