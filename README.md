# ClaudeCode-Hono

This repository contains **no application code**. It is a portable [Claude Code](https://code.claude.com) configuration preset that teaches Claude Code how to work on a **Hono + Prisma + Better Auth** API (`OpenAPIHono`, Zod validation via `@hono/zod-openapi`, Prisma ORM, Better Auth for authentication, Vitest for testing).

Drop the contents of this repo into the root of a real project and Claude Code will automatically enforce these conventions on every task, without needing to re-explain them each session.

## What's inside

```
CLAUDE.md                  # Root rules file — stack, project structure, hard "never do" list
.claude/
├── rules/                 # Detailed per-topic rules, @imported from CLAUDE.md
│   ├── hono.md             # No extracted handlers, createMiddleware(), typed context, HTTPException, no app.head()
│   ├── openapi.md           # createRoute()/app.openapi() only, schema/route/router file ordering
│   ├── prisma.md            # Singleton client, schema conventions, mandatory pagination
│   ├── auth.md               # Better Auth singleton, bearer/admin plugins, role middleware order
│   ├── testing.md            # Vitest conventions, Prisma/Better Auth mocking, required coverage
│   └── typescript.md         # strict mode, no `any`, required env vars
└── skills/                 # Better Auth reference skills (installed via the `skills` CLI, see below)
    ├── better-auth-best-practices/
    ├── better-auth-security-best-practices/
    ├── create-auth/
    ├── email-and-password-best-practices/
    └── two-factor-authentication-best-practices/
skills-lock.json            # Lockfile pinning the exact source/commit/hash of each installed skill
```

`CLAUDE.md` is the entry point: it defines the stack, the project layout, the "Adding a New Resource" checklist, and a list of things Claude must never do (hand-rolled auth, manual Prisma instantiation, `app.get()`/`app.post()` for API routes, etc.). It then `@imports` each file in `.claude/rules/` so the full rule set loads automatically — you never need to reference the sub-files manually.

The `.claude/skills/` directory holds [Agent Skills](https://code.claude.com/docs/en/skills) sourced from the [`better-auth/skills`](https://github.com/better-auth) repository. Claude loads a skill's `SKILL.md` on demand when a task matches its description (e.g. asking about 2FA pulls in `two-factor-authentication-best-practices`), instead of keeping that context loaded at all times.

## Installing this config into a project

### 1. Copy the config files

From this repo's root, copy the config into your target project (a Node.js project using Hono + Prisma):

```bash
cp CLAUDE.md /path/to/your-project/
cp -r .claude /path/to/your-project/
cp skills-lock.json /path/to/your-project/
```

If the target project already has a `CLAUDE.md` or `.claude/` directory, merge instead of overwriting — check for conflicting rules first.

### 2. Restore the skills from the lockfile

Skills are managed with the [`skills` CLI](https://www.skills.sh) (`npx skills`), not committed as vendored copies you're expected to hand-edit. `skills-lock.json` pins the exact source and content hash for each one, so re-fetch them rather than trusting the copies you pasted in step 1:

```bash
npx skills experimental_install
```

This reads `skills-lock.json` from the project root and re-downloads each skill at its pinned commit/hash. (As of this writing `experimental_install` is the lockfile-restore command exposed by the `skills` CLI — check `npx skills --help` if it has since been renamed to `skills install`.)

To add a skill that isn't in the lockfile yet:

```bash
npx skills add better-auth/skills
```

### 3. Verify Claude Code picks it up

From the project root:

```bash
claude
```

Ask Claude something like *"what are the rules for adding a new route?"* — it should answer from `CLAUDE.md` / `.claude/rules/openapi.md` without you pasting anything. You can also run `/context` inside a Claude Code session to confirm `CLAUDE.md` and the imported rule files are loaded.

### 4. Adapt to your project

The rules assume a specific project layout (`src/lib/prisma.ts`, `src/lib/auth.ts`, `src/routes/`, `src/tests/`, etc. — see the **Project Structure** section of `CLAUDE.md`). If your project's layout differs, edit `CLAUDE.md` and the relevant file(s) in `.claude/rules/` to match before relying on Claude to follow them — Claude enforces whatever the files say, not the intent behind them.

## Updating the rules

- Edit `CLAUDE.md` for stack-level or structural changes; edit the relevant file in `.claude/rules/` for a single topic (auth, Prisma, testing, etc.).
- Keep the `@.claude/rules/*.md` import list at the bottom of `CLAUDE.md` in sync if you add or remove a rules file.
- Re-run `npx skills experimental_install` after editing `skills-lock.json` by hand, or use `npx skills add`/`skills remove` rather than editing the lockfile directly.
