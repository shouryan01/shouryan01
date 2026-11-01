# Web tech stack standard

Last reviewed: November 1, 2026.

Companion standards: [Tauri desktop](tauri-tech-stack.md) and [shared web/desktop codebases](web-desktop-tech-stack.md).

Use this document as the starting point for new web projects. Required defaults apply when the relevant capability is needed; conditional tools are added only when the project needs them. Record exceptions and their reasons in the project's README or architecture decisions. This standard does not require retrofitting existing projects.

## Required defaults

| Area | Standard |
| --- | --- |
| Language | TypeScript with strict checking |
| Frontend | React |
| Toolchain | Vite+: Vite, Rolldown, Oxlint, Oxfmt, Vitest, and integrated Go-powered TypeScript checking |
| Environment | Global Vite+ CLI manages Node.js and pnpm |
| Package manager | pnpm; commit `pnpm-lock.yaml` |
| Styling | Tailwind CSS and shadcn/ui |
| Validation | Zod |
| SPA routing | TanStack Router only |
| Server rendering | TanStack Start |
| Database on Cloudflare | Cloudflare D1 when persistence is required |
| ORM | Drizzle, with reviewed, versioned migrations |
| Deployment | Cloudflare by default; Vercel when project requirements favor it |

Choose the latest stable, mutually compatible releases at project creation. Use Vite+'s supported bundled tool versions instead of independently forcing newer Vite, Rolldown, or Vitest versions. Check framework and deployment-plugin compatibility before adopting an upgrade. Do not make prereleases the default.

Pin exact Node.js and pnpm versions per project, using a supported Node.js LTS release. Commit the version declarations and lockfile; use frozen-lockfile installs in CI. Re-evaluate versions when starting each project rather than treating this document as a permanent version list.

### Vite+, Volta, and type checking

Vite+ has two distinct parts: the global `vp` CLI manages Node.js and package-manager versions, while the project-local `vite-plus` dependency provides the development toolchain. The global environment manager replaces Volta for Node.js and pnpm selection in this standard. Installing only the local package does not provide environment management. Avoid competing environment managers controlling the same project. See the [Vite+ overview](https://viteplus.dev/guide/) and [environment guide](https://viteplus.dev/guide/env).

Use `vp env pin` to save project versions and `vp env current` to inspect the selected environment. If pnpm also attempts to manage runtimes or its own version, follow the environment guide's settings for the pinned pnpm major so Vite+ remains the owner.

Enable both `lint.options.typeAware` and `lint.options.typeCheck` in `vite.config.ts`. This makes `vp check` run formatting, linting, and Go-powered TypeScript checking through Oxlint's type-aware path. It satisfies the default need for tsgo-based checking without a separate `tsgo` command. Check advanced TypeScript requirements against the selected release. See [Vite+ checks](https://viteplus.dev/guide/check).

Use the standard project workflow after setup:

```sh
vp install
vp dev
vp check
vp test
vp build
```

`vp dev` starts the development server; run checks, tests, and builds separately from it. CI must run noninteractive tests and a frozen-lockfile install using the project's pinned tooling. When the global CLI is unavailable but the pinned Node.js and pnpm are installed, invoke local tooling with commands such as `pnpm exec vp check`. See the [Vite+ getting-started guide](https://viteplus.dev/guide/).

## Conditional capabilities

| Need | Default |
| --- | --- |
| Authentication | Better Auth |
| Separate backend API | Hono |
| Remote state, caching, and mutations | TanStack Query |
| Advanced tables | TanStack Table with shadcn/ui presentation |
| Complex forms | TanStack Form with Zod validation |
| Unit and integration tests | Vitest through Vite+ |
| Critical browser flows | Playwright |

Prefer an appropriate TanStack capability or shadcn/ui component before adding a competing library or building equivalent infrastructure. These are preferred tools, not a requirement to install the entire ecosystem. Use React's local state for simple UI state and native form controls where sufficient.

### Routing, data, and APIs

- Use TanStack Router as the only application router for SPAs. Use TanStack Start for server-rendered applications; it builds on TanStack Router.
- Choose rendering based on public content, SEO, metadata, and application needs. Do not require server rendering for every application.
- Use TanStack Start server functions for application backend work when sufficient. Add Hono when a separate API is required, such as an API serving additional clients.
- Define request/response contracts and consistent errors. Validate external inputs with Zod on the server; client validation improves usability but does not replace server validation or authorization.
- Give each remote data flow a clear fetching and caching owner. Coordinate Router loaders and Query rather than maintaining duplicate caches or fetching the same data independently.

References: [TanStack Router](https://tanstack.com/router/latest/docs/framework/react/overview), [TanStack Start](https://tanstack.com/start/latest/docs/framework/react/overview), [Query](https://tanstack.com/query/latest/docs/framework/react/overview), [Table](https://tanstack.com/table/latest/docs/introduction), [Form](https://tanstack.com/form/latest/docs/overview), [Zod](https://zod.dev/), and [Hono on Cloudflare Workers](https://hono.dev/docs/getting-started/cloudflare-workers).

### Authentication and persistence

- Add Better Auth only when authentication is needed. Choose login methods, authorization rules, account recovery/deletion, and session behavior explicitly. Enforce access rules on the server.
- Use Drizzle for application persistence. Select the driver and schema dialect for the actual database and deployment runtime; verify Better Auth adapter compatibility as part of setup.
- Default to D1 for Cloudflare projects. For Vercel or another runtime, explicitly select the database and provider rather than assuming a D1 Worker binding is available there.
- D1 uses SQLite semantics; PostgreSQL is a different database target. Switching requires deliberate schema, SQL, driver, data migration, and operational changes. Drizzle does not make the engines interchangeable.
- Generate and review migrations, commit them, and apply them through one documented migration workflow. Verify the selected Drizzle migration layout and runner work together. Test migrations before production; avoid unreviewed schema pushes to production.
- Separate local, preview, and production databases. Document backup/restore procedures and recovery expectations before storing important data.

References: [Better Auth installation and adapters](https://better-auth.com/docs/installation), [Drizzle with D1](https://orm.drizzle.team/docs/sqlite/connect-cloudflare-d1), and [D1 migrations](https://developers.cloudflare.com/d1/reference/migrations/).

### Styling and deployment

Use Tailwind's Vite integration and shadcn/ui's current framework installation instructions. Establish shared theme tokens and reuse components. Require accessible controls, keyboard navigation, responsive layouts, and explicit loading, empty, and error states.

Choose the deployment runtime early. Use the supported Cloudflare integration for Workers-based TanStack Start applications, or Vercel's supported deployment path when selected. Confirm dependency compatibility with the target runtime; local Node.js tooling does not imply production code runs in Node.js. For SPAs, verify direct navigation to application routes works on the host.

References: [Tailwind with Vite](https://tailwindcss.com/docs/installation/using-vite), [shadcn/ui with Vite](https://ui.shadcn.com/docs/installation/vite), [TanStack Start on Cloudflare](https://developers.cloudflare.com/workers/framework-guides/web-apps/tanstack-start/), and [TanStack Start on Vercel](https://vercel.com/kb/guide/deploy-a-tanstack-start-app-to-vercel).

## Quality and maintenance

- Require CI to pass formatting, linting, type checks, relevant tests, and a production build. Enable the integrated type checks explicitly; a successful build alone is not type-check evidence.
- Test behavior with Vitest, including important API validation, authorization, and database interactions when present. Use Playwright for critical journeys such as sign-in and core user actions. Include keyboard and responsive-layout checks.
- Validate migrations and a deployment preview against the actual target runtime before release. Cover SPA deep links or server-rendered routes as appropriate.
- Document required environment variables in a secret-free example file. Keep credentials out of source control and browser-exposed configuration; separate local, preview, and production secrets.
- Review dependency updates regularly, read migration notes, and require project checks to pass. Revisit this standard when tooling or compatibility guidance changes.

Testing references: [Vitest](https://vitest.dev/guide/) and [Playwright](https://playwright.dev/docs/intro).

## Project kickoff checklist

Record these decisions in the new project's documentation. Mark unused capabilities as not applicable; select additional service providers only when needed.

- [ ] Rendering: SPA or TanStack Start; SEO, public metadata, and indexing requirements.
- [ ] Runtime and hosting: Cloudflare or Vercel; compatibility, preview deployments, domains, and release/rollback process.
- [ ] Data: database engine/provider, Drizzle driver, schema ownership, migration runner, environment isolation, backups, and recovery.
- [ ] Identity: whether auth is needed, login methods, roles/permissions, account lifecycle, and sessions.
- [ ] API: whether Hono is needed, consumers, contracts, validation, errors, and applicable request limits.
- [ ] UI and state: theme, component reuse, local versus remote state, forms, and tables.
- [ ] Optional services: uploads/storage, transactional email, payments, scheduled work, and background jobs.
- [ ] Operations: logging, error reporting, analytics, privacy/data retention, and expected costs or usage limits.
- [ ] Audience: supported browsers, accessibility, localization, and time-zone handling.
- [ ] Repository: single package by default; pnpm workspace when multiple apps or shared packages justify it.
- [ ] Delivery: exact runtime/package-manager pins, lockfile, CI checks, test coverage for critical behavior, and environment-variable documentation.

This document defines project defaults, not a scaffold. Consult the linked official instructions when creating a project, and record any compatibility-driven exceptions.
