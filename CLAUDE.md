# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is the **shadcn/ui** monorepo — a collection of accessible, customizable React components that users copy-paste into their own projects. It is NOT a component library distributed as an npm package of components; instead, it ships a CLI (`shadcn`) that installs individual components into a user's project.

The monorepo uses **pnpm workspaces** + **Turborepo** and contains:

- `apps/www` — The documentation website (Next.js 14, port 3333). Hosts the component registry at `/r`.
- `apps/v4` — The Tailwind v4 / React 19 variant of the registry (Next.js 15, Turbopack).
- `packages/shadcn` — The `shadcn` CLI package (published to npm as `shadcn`).
- `packages/cli` — Legacy CLI package (`shadcn-ui`); superseded by `packages/shadcn`.
- `templates/` — Starter templates synced via `scripts/sync-templates.sh`.

## Common Commands

```bash
pnpm install                  # Install all dependencies

# Development
pnpm www:dev                  # Run docs site on http://localhost:3333
pnpm v4:dev                   # Run v4 site (Turbopack)
pnpm shadcn:dev               # Build CLI in watch mode

# Running CLI locally (against local registry)
pnpm www:dev                  # Start registry first (port 3333)
pnpm shadcn:dev               # In another terminal — rebuilds CLI on change
pnpm shadcn                   # Run CLI against local registry
pnpm shadcn add button -c ~/Desktop/my-app   # Test CLI in a target app

# Testing
pnpm test                     # Run all tests (Vitest, excludes shadcn-ui legacy)
pnpm --filter=shadcn test     # Run CLI tests only
pnpm shadcn:test              # Same as above

# Registry
pnpm build:registry           # Rebuild registry JSON (runs for both www and v4)

# Quality
pnpm lint                     # Lint all workspaces
pnpm lint:fix                 # Auto-fix lint issues
pnpm typecheck                # TypeScript check all workspaces
pnpm format:write             # Format all files with Prettier
pnpm check                    # Run lint + typecheck + format:check together
```

## Architecture

### Registry System

The registry is the central concept. It defines every component, block, hook, and lib item in a typed schema (`packages/shadcn/src/registry/schema.ts`).

**How components flow:**

1. **Source of truth**: `apps/www/registry/new-york/ui/*.tsx` (the `new-york` style is canonical; `default` style is auto-synced from it during registry build).
2. **Registry index**: `apps/www/registry/index.ts` assembles all items (ui, blocks, charts, lib, hooks, themes, examples, internal) into a single `Registry` object.
3. **Build step**: `pnpm build:registry` runs `apps/www/scripts/build-registry.mts` which:
   - Syncs `new-york` → `default` style directories
   - Emits JSON files to `apps/www/public/r/` (served as the live registry API)
   - Populates `apps/www/__registry__/` (auto-generated, do not edit)
4. **CLI install**: When a user runs `shadcn add button`, the CLI fetches from the registry URL and applies transformers before writing to the user's project.

### Two Styles

Components exist in two style variants:
- `new-york` — the primary style, edited by hand
- `default` — synced automatically from `new-york` during `build:registry`; do not edit directly

When modifying or adding a component, **only edit the `new-york` version**, then run `pnpm build:registry` to propagate.

### v4 Registry

`apps/v4` maintains a parallel registry for **Tailwind CSS v4** (uses `@tailwindcss/postcss`, CSS-first config, no `tailwind.config.js`). Its components live under `apps/v4/registry/new-york-v4/`. The v4 registry is built separately via `pnpm --filter=v4 registry:build`.

### CLI Package (`packages/shadcn`)

Entry: `packages/shadcn/src/index.ts` → Commander CLI with subcommands:
- `add` — fetch and install components; resolves `registryDependencies` recursively
- `init` — scaffold `components.json` and initial config in a user's project
- `diff` — show updates available for installed components
- `build` — build a custom registry
- `migrate` — migrate icon libraries

**Transformer pipeline** (`packages/shadcn/src/utils/transformers/`): When components are installed, source files pass through a series of AST transforms (via `ts-morph` + Recast):
- `transform-import` — rewrites `@/` aliases to match the user's config
- `transform-rsc` — adds/removes `"use client"` directives
- `transform-css-vars` — inlines CSS variables if user opted out of CSS vars
- `transform-tw-prefix` — adds Tailwind prefix if configured
- `transform-icons` — rewrites icon imports to the user's chosen library

**Config** (`components.json` in user's project): Read by `packages/shadcn/src/utils/get-config.ts`. Tracks style, Tailwind settings, aliases, and icon library.

### Documentation Site (`apps/www`)

- Next.js app router with route groups: `(app)` for public pages, `(internal)` for dev-only pages.
- **MDX docs** in `apps/www/content/docs/` processed by **Contentlayer2**.
- Component examples are embedded in docs via the `rehype-component` plugin, which reads files from `__registry__/` (auto-generated).
- The site runs on port **3333** so the local CLI's `REGISTRY_URL=http://localhost:3333/r` resolves correctly.

## Commit Convention

Follows Conventional Commits (`@commitlint/config-conventional`):

```
feat(components): add new prop to avatar component
fix(cli): handle missing tsconfig gracefully
docs: update theming guide
build: upgrade tailwindcss to 3.4.6
```

Categories: `feat`, `fix`, `refactor`, `docs`, `build`, `test`, `ci`, `chore`.

## Adding or Modifying Components

1. Edit/create the component in `apps/www/registry/new-york/ui/` (and `new-york/examples/`, `new-york/blocks/` as needed).
2. Register it in the appropriate registry file (`registry-ui.ts`, `registry-blocks.ts`, etc.) with its `name`, `type`, `dependencies`, `registryDependencies`, and `files`.
3. If schema changes are made to `packages/shadcn/src/registry/schema.ts`, also update `apps/www/public/schema/registry-item.json`.
4. Run `pnpm build:registry` — this syncs `default` style and generates the public JSON.
5. Add/update the MDX doc in `apps/www/content/docs/components/`.

## Testing the CLI

The CLI tests live in `packages/shadcn/test/` and `packages/shadcn/src/**/*.test.ts`. They use Vitest and MSW for mocking registry HTTP responses. Run against local registry with:

```bash
pnpm --filter=shadcn test:dev   # REGISTRY_URL=http://localhost:3333/r
```

## Code Style

- **Prettier**: no semicolons, double quotes, 2-space indent, LF line endings, trailing commas (ES5). Import order is enforced via `@ianvs/prettier-plugin-sort-imports`.
- **TypeScript**: strict mode. CLI uses `ts-morph` for AST operations on user code; avoid raw string manipulation for code transforms.
- **Zod** is used pervasively for runtime validation of registry schemas and CLI config.
