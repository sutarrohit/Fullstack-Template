# CLAUDE.md — apps/server

Hono API (`@repo/api`) for the Turborepo template: Zod OpenAPI routes, Prisma 7 on PostgreSQL, better-auth, deployable to AWS Lambda via CDK.

## Commands (run in `apps/server`)

```sh
cp .env.example .env          # DATABASE_URL, DIRECT_URL, BETTER_AUTH_SECRET (>=16 chars), PUBLIC_URL are required
pnpm dev                      # tsx watch, http://localhost:4000 — Swagger at /docs, spec at /doc, /health
pnpm build                    # tsc → dist/
pnpm test                     # vitest run (tests load .env.test via NODE_ENV=test)
pnpm exec vitest run tests/unit/demo.test.ts      # single file
pnpm exec vitest run -t "confirms the test"        # single test by name

pnpm db:generate              # prisma generate → prisma/generated (run after every schema change)
pnpm db:migrate               # prisma migrate dev
pnpm db:seed                  # / db:seed:reset
pnpm cdk:diff | cdk:deploy    # Lambda deploy, see deploy.md
```

`src/env.ts` validates env with Zod and **exits the process** if anything is missing, so every entry point (tests included) needs a complete `.env` / `.env.test`.

## Conventions

- ESM (`"type": "module"`): relative imports carry a `.js` suffix (`'../lib/api-error.js'`). The `@/*` alias maps to the package root (`@/prisma/generated/client.js`, `@/src/...`).
- Prisma client is generated into `prisma/generated/` and imported from there, not from `@prisma/client`. The single `PrismaClient` (with the `PrismaPg` adapter) is in `src/lib/prisma.ts`.
- Services are classes with constructor-injected `PrismaClient`; instances are wired once in `src/lib/container.ts`. Handlers import from the container, never construct services.
- Expected failures: `throw new ApiError(status, 'CODE', message)` (`src/lib/api-error.ts`). `on-error.middleware.ts` turns any error into `{ statusCode, message, stack }` (stack omitted in production).
- Status codes come from `stoker/http-status-codes`; response bodies are declared with `jsonContent(schema, description)` from `stoker/openapi/helpers`.
- `AppBinding` in `src/types/index.ts` types `c.get('user')` and `c.get('logger')`.

## Adding a route

Mirror `src/routes/user/`:

1. `<name>.route.ts` — `createRoute({ method, path, tags, request?, responses })` with Zod schemas from `@hono/zod-openapi`.
2. `<name>.handler.ts` — `AppRouteHandler<typeof xRoute>` functions; thin, delegate to a service.
3. `<name>.index.ts` — `createRouter().openapi(route, handler)...`, default export.
4. Register the router in the `routes` array in `src/app.ts` (this also feeds `AppType` for the typed `hc` / `testClient`).
5. Business logic in `src/services/<name>.service.ts`, instantiated in `src/lib/container.ts`.

Integration tests use `testClient(app)` from `tests/helper/client.ts`.

## Known gaps in the template

Verify before relying on these; fix them when you touch the area:

- `app.ts` mounts routers at `/`, so routes are currently served without the `/api/v1/<name>` prefix that comments, the web client (`apps/web/utils/request.ts`) and tests assume.
- `requireAuth` (`src/middlewares/auth.middleware.ts`) is not applied to any router, and the better-auth handler (`auth.handler`) is not mounted — handlers that call `c.get('user')` get `undefined`.
- `src/index.ts` hardcodes port 4000 instead of using `env.PORT`.
- `UserService.getOnboardingStatus` returns `completed: true` for any existing user instead of checking `onboardingCompletedAt`.

## Deploy

- Docker: `docker compose up --build` (build context is the repo root because of `@repo/*` workspace deps).
- Lambda: `lambda/index.ts` wraps the app with `hono/aws-lambda`; `infra/stack.ts` is the CDK stack. AWS/CDK packages are devDependencies and must stay out of the runtime image.
