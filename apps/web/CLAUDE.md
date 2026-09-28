# CLAUDE.md — apps/web

Next.js frontend of the Turborepo template. Talks to the Hono API in `apps/server` (or the FastAPI server in `packages/fastapi-server`).

## Commands

Install from the **repo root** (`pnpm install`). The `pnpm-workspace.yaml` / `pnpm-lock.yaml` inside `apps/web` are leftovers from `create-next-app`; the root workspace is the one that counts.

```sh
pnpm dev                         # next dev on :3000 (run in apps/web)
pnpm build && pnpm start
pnpm lint                        # eslint (next core-web-vitals + typescript)
pnpm exec tsc --noEmit           # typecheck — there is no check-types script yet
```

From the root: `pnpm turbo dev --filter=apps` (the package is named `apps`, not `web`).

## Stack

- Next.js 16 App Router, React 19, **React Compiler on** (`next.config.ts`) — don't add `useMemo`/`useCallback` just for memoization.
- Tailwind CSS v4: no `tailwind.config`; theme tokens and `@theme inline` live in `app/globals.css`.
- shadcn/ui, style `base-nova`, built on **Base UI (`@base-ui/react`), not Radix**. Custom triggers use the `render` prop, not `asChild`. Icons: `@remixicon/react`.
- TanStack Query v5, `next-themes`, `sonner`, `recharts`, Zod v4.
- Path alias `@/*` → `apps/web/*`.

## shadcn

The `shadcn` skill is installed in this app (`apps/web/.claude/skills/shadcn`, pinned in `skills-lock.json`). Run the CLI **from `apps/web`** — that's where `components.json` is:

```sh
pnpm dlx shadcn@latest add <component>
pnpm dlx shadcn@latest docs <component>
```

- Generated components go to `components/ui/`. Treat them as vendored: compose and use variants rather than editing them.
- Use semantic tokens (`bg-background`, `text-muted-foreground`, `bg-primary`), not raw palette colors or manual `dark:` overrides. `app/page.tsx` is a throwaway demo and doesn't follow this.
- `cn()` from `@/lib/utils` for conditional classes.

## Layout

```
app/                 layout.tsx (fonts, <Providers>), page.tsx, globals.css
components/ui/       shadcn components
providers/           QueryClientProvider + ThemeProvider (client component)
lib/getQueryClient.ts  one QueryClient per request on the server, singleton in the browser
lib/api/<domain>/    <domain>-apis.ts (fetch functions) + <domain>-queries.ts (queryOptions/mutationOptions)
utils/request.ts     fetch wrapper: prefixes `${NEXT_PUBLIC_API_URL}/api/v1`, JSON headers
utils/handleResponse.ts  204 → undefined; non-2xx → throws Error(data.message)
env.ts               Zod-validated env (NEXT_PUBLIC_API_URL, default http://localhost:4000)
hooks/               use-mobile.ts
```

## Adding an API call

Follow `lib/api/user/`:

1. `lib/api/<domain>/<domain>-apis.ts` — typed functions calling `request("/path", { method })`. Paths are relative to `/api/v1`.
2. `lib/api/<domain>/<domain>-queries.ts` — `queryOptions({ queryKey: [<domain>, ...], queryFn })` / `mutationOptions(...)`.
3. Components use `useQuery(xQueryOptions())` / `useMutation(xMutationOptions())`.

New env vars go into the schema in `env.ts` and must be read explicitly from `process.env.NAME` there (Next inlines `NEXT_PUBLIC_*` only on literal access).

## Style

Prettier config is at the repo root (`.prettierrc`); run `pnpm format` from the root.
