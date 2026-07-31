# TypeScript

- `strict: true` in `tsconfig.json` — no exceptions.
- No `any` unless wrapping an external untyped value at a boundary (e.g. Prisma mock casting). When `any` is unavoidable, add a comment explaining why.
- All exported functions and middleware have explicit return types or rely on correct Hono generics for inference. Never widen a type to avoid a TypeScript error — fix the type instead.

---

# Environment Variables

All environment variables are accessed via `process.env`. Required variables:

| Variable       | Used by              | Required |
|---------------|----------------------|----------|
| `DATABASE_URL` | Prisma               | Yes      |
| `JWT_SECRET`   | `hono/jwt`           | Yes      |
| `PORT`         | `src/index.ts`       | No (default 3000) |
| `CORS_ORIGIN`  | CORS middleware      | No (default `*`) |
| `NODE_ENV`     | Prisma log config    | No       |

Never hardcode secrets. Never commit `.env`. The test setup sets `JWT_SECRET=test-secret` in `src/tests/setup.ts` — do not rely on a `.env` file in tests.
