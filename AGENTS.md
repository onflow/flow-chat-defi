# AGENTS.md

This file provides guidance to AI coding agents (Claude Code, Codex, Cursor, Copilot, and others)
when working in this repository. It is loaded into agent context automatically — keep it concise.

## Overview

Flow Chat is a DeFi and onchain analytics chatbot for the Flow ecosystem, built on Vercel's
Chat SDK. It runs as a Next.js 15 App Router application that talks to the FLOW-MCP server
over SSE for Flow / Flow-EVM tooling and uses the Vercel AI SDK to route requests across
Anthropic, OpenAI, and xAI models. EVM wallet connection is handled with wagmi + RainbowKit;
`@onflow/fcl` is available for Flow-mainnet interactions.

## Build and Test Commands

Package manager is pnpm (`packageManager: pnpm@9.12.3` in `package.json`).

- `pnpm install` — install deps
- `pnpm dev` — start Next.js dev server with Turbopack
- `pnpm build` — runs `tsx lib/db/migrate` then `next build` (migrations run on every build)
- `pnpm start` — serve the production build
- `pnpm lint` / `pnpm lint:fix` — `next lint` + `biome lint --write --unsafe`
- `pnpm format` — `biome format --write`
- `pnpm test` — sets `PLAYWRIGHT=True` and runs Playwright (`e2e` and `routes` projects)

### Database (Drizzle)

- `pnpm db:generate` — generate SQL from `lib/db/schema.ts`
- `pnpm db:migrate` — apply migrations (`lib/db/migrate.ts`)
- `pnpm db:studio` / `pnpm db:push` / `pnpm db:pull` / `pnpm db:check` / `pnpm db:up`

Migrations live in `lib/db/migrations/` and target Postgres (`drizzle.config.ts` —
`dialect: 'postgresql'`, `POSTGRES_URL`).

## Architecture

- `app/(auth)/` — NextAuth v5 beta routes (`auth.ts`, `auth.config.ts`, `login`, `register`,
  `api/auth`). Guest auth is the default entry point via `/api/auth/guest`.
- `app/(chat)/` — chat UI and APIs: `api/chat`, `api/document`, `api/files`, `api/history`,
  `api/suggestions`, `api/vote`. Root `page.tsx` is the chat landing page.
- `middleware.ts` — redirects unauthenticated requests to the guest endpoint; exposes `/ping`
  for Playwright health checks; excludes Next static/image assets.
- `lib/ai/` — Vercel AI SDK plumbing. `providers.ts` wires Anthropic / OpenAI / xAI models;
  `models.ts` + `models.test.ts`; `prompts.ts`; `entitlements.ts`; `tools/` contains
  `create-document.ts`, `find-flow-doc.ts`, `flow-blcok-height.ts`, `get-weather.ts`,
  `request-suggestions.ts`, `update-document.ts`.
- `lib/db/` — Drizzle schema, migrations, queries, helpers.
- `artifacts/` — artifact kinds (`code/`, `image/`, `sheet/`, `text/`), each with
  `client.tsx` and `server.ts`.
- `components/` — React components. `components/ui/` holds shadcn/ui primitives;
  `web3-provider.tsx` sets up wagmi + RainbowKit.
- `hooks/` — custom React hooks (artifact state, chat visibility, transaction listener, etc.).
- `tests/` — Playwright tests; `tests/e2e/` and `tests/routes/` map to the two Playwright
  projects in `playwright.config.ts`.

### External services

- FLOW-MCP over SSE at `https://flow-mcp-production.up.railway.app/sse` (see
  `app/(chat)/api/chat/route.ts`). Tools are loaded at request time via
  `experimental_createMCPClient`.
- Vercel Postgres (`@vercel/postgres`), Vercel Blob (`@vercel/blob`), Redis (`redis`),
  OpenTelemetry via `@vercel/otel` (`instrumentation.ts`).

## Conventions and Gotchas

- **Path alias:** `@/*` maps to the repo root (`tsconfig.json`). Use `@/components/...`,
  `@/lib/...`, etc.
- **Formatter:** Biome is the formatter/linter (`biome.jsonc`). Single quotes, semicolons,
  trailing commas, 80-col lines, 2-space indent. `organizeImports` is disabled. ESLint still
  runs for Next/Tailwind rules (`.eslintrc.json`).
- **Biome ignores:** `lib/db/migrations`, `lib/editor/react-renderer.tsx`, `node_modules`,
  `.next`, `public`, `.vercel`. ESLint ignores `**/components/ui/**`.
- **`pnpm build` runs migrations.** `POSTGRES_URL` must be set or the build fails at
  `tsx lib/db/migrate`.
- **Test env:** `pnpm test` sets `PLAYWRIGHT=True`. `lib/constants.ts` exposes
  `isTestEnvironment` / `isDevelopmentEnvironment` — `lib/ai/providers.ts` swaps in mock
  models when `isTestEnvironment` is true.
- **Dev server for Playwright:** `playwright.config.ts` boots `pnpm dev` and waits on
  `/ping` (see `middleware.ts`). Playwright projects: `e2e` (`tests/e2e/*.test.ts`) and
  `routes` (`tests/routes/*.test.ts`).
- **Auth:** NextAuth 5 beta. Unauthenticated visitors are redirected to
  `/api/auth/guest?redirectUrl=...`; `guestRegex` in `lib/constants.ts` identifies guest
  emails.
- **Next experimental flags:** PPR is enabled (`next.config.ts` — `experimental.ppr: true`).
  Remote image hostnames are allow-listed in `next.config.ts`; add new hosts there.
- **AI providers:** provider IDs are defined in `lib/ai/providers.ts` (`AIProviderType`).
  Default is `claude-3-5-haiku`. Add new models by extending `providerModels` plus
  `AIProviderType`.
- **shadcn/ui:** base color `zinc`, RSC enabled, aliases defined in `components.json`. New
  primitives go in `components/ui/`.
- **No Cadence contracts live here.** `@onflow/fcl` is a client dependency; all Flow /
  Flow-EVM chain logic is delegated to FLOW-MCP tools.

## Files Not to Modify

- `lib/db/migrations/` — generated by Drizzle Kit; regenerate via `pnpm db:generate`.
- `components/ui/` — shadcn primitives (ESLint-ignored; regenerate via the shadcn CLI).
- `pnpm-lock.yaml`, `next-env.d.ts`.
