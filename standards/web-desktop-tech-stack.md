# Shared web and desktop tech stack standard

Last reviewed: November 1, 2026.

Use this standard when one product ships a browser application and a Tauri desktop application. Inherit the [web standard](tech-stack.md) and [Tauri standard](tauri-tech-stack.md). This document defines how they fit together; it takes precedence for sharing and platform boundaries. Record exceptions in the project's architecture decisions.

## Architecture defaults

| Area | Standard |
| --- | --- |
| Repository | Single application package with one `package.json` and pnpm lockfile; no monorepo |
| Browser application | React SPA with Vite+ and TanStack Router by default |
| Desktop application | Tauri with a bundled React SPA, Vite+, TanStack Router, and Rust |
| Shared code | UI, theme, domain logic, Zod contracts, API client, and reusable features |
| Platform access | Explicit interfaces with browser and desktop implementations |
| Remote API | Hono when required to serve both clients |
| Remote identity | One Better Auth service when authentication is required |
| Remote persistence | Drizzle; D1 by default on Cloudflare |
| Desktop persistence | Rust-owned SQLite when durable local storage is required |
| Delivery | Web/backend to Cloudflare by default or Vercel; desktop installers via native release CI |

Use the latest stable, compatible dependencies and pinned environments from the companion standards. Keep shared code in ordinary `src/` modules. Use Vite+ scripts and the Tauri CLI; no workspace packages or monorepo task orchestrator are required.

### Single-project layout

```text
src/                   # One React application for browser and desktop
  components/          # Shared UI and theme
  features/            # Product features and domain logic
  routes/              # TanStack Router routes
  lib/                 # Zod contracts and remote API client
  platform/            # Browser/native adapters and runtime selection
src-tauri/             # Rust, Cargo.lock, capabilities, and native configuration
public/                # Static frontend assets
index.html             # Vite frontend entry
package.json           # One application package and scripts
pnpm-lock.yaml
vite.config.ts
```

These `src/` subfolders are conventions, not required abstractions. Keep one frontend entry point and route tree. Add platform adapters only where behavior differs. An optional Hono backend may live in a server-only `server/` directory with its own deployment configuration, using the same package; exclude it from frontend imports. A separately maintained API can also serve the app. Neither choice requires a workspace.

### Build and deployment

The same frontend source can run in a browser and inside Tauri. Deploy the compiled frontend output, normally `dist/`, rather than uploading raw `src/`. Vite also needs the root `index.html`, configuration, dependencies, and public assets to produce that output. See [Vite production builds](https://vite.dev/guide/build.html).

- Web: run `vp build` from the project root and publish `dist/` to Cloudflare or Vercel. Configure SPA route fallback and verify direct navigation to nested routes.
- Desktop: run `pnpm exec tauri build`. Configure Tauri's frontend build hook to run the same frontend build and set `frontendDist` to `../dist` from `src-tauri/tauri.conf.json`. Tauri bundles those assets into the native application.
- Development: `vp dev` runs the browser frontend; `pnpm exec tauri dev` runs it inside the native shell through the configured development hook.

Start with one frontend build configuration. Use target-specific modes only when needed for API URLs, asset paths, or excluding native adapters. Validate base paths and routing in both deployments. Web deployment requires no Rust build when all required browser behavior is available in the frontend or remote API. See [Tauri with Vite](https://v2.tauri.app/start/frontend/vite/).

### Rendering and sharing

Use one React SPA and TanStack Router for this combined application. Keep platform selection in `src/platform/` and reuse components, routes, contracts, and features directly.

Tauri requires bundled static assets and does not natively host an SSR server. If the product later needs TanStack Start SSR, revisit the architecture explicitly rather than applying an SSR build to the desktop shell. A separate public site may be appropriate, but is not part of this default. See [Tauri frontend configuration](https://v2.tauri.app/start/frontend/).

## Platform and data boundaries

- Define small platform interfaces only for needed capabilities, such as file selection, opening external URLs, credentials, and notifications. Supply implementations at app initialization; shared features consume interfaces rather than scattered runtime checks.
- Keep Tauri imports and `invoke` wrappers in native adapter modules. Select the adapter at runtime using Tauri detection (such as `isTauri()`), and load native implementations lazily. Do not invoke native APIs during shared module initialization. A stricter target-specific build can exclude native adapters entirely when needed; browser safety does not require duplicating the app.
- Represent unsupported browser capabilities explicitly: provide an appropriate browser implementation or disable the feature with a clear explanation. Avoid silently dropping user actions.
- Keep one remote API client and shared Zod contracts, with platform-specific authentication/transport supplied by each app. Keep backend implementation, database clients, and server secrets out of all client bundles. TypeScript types alone do not validate network data.
- Use TanStack Query when remote caching is needed; keep simple UI state in React. Define query invalidation and account-change cleanup consistently across clients.
- Treat web and desktop clients as independent API consumers. Desktop releases can lag behind the deployed web app, so make compatible API changes by default and define a minimum supported client version when a breaking change is unavoidable.

### Authentication

Use one Better Auth backend and authorization model when accounts are shared. Use the browser session mechanism for web; implement and test the desktop browser-to-app callback and secure credential handling described in the Tauri standard. Never assume a cookie in the system browser is already available inside Tauri. Validate access on the server regardless of the client.

Configure browser CORS/cookie behavior and desktop origins/redirects explicitly for development and production. Test login, expired sessions, logout, revoked access, and account switching in both clients. Select any desktop-specific auth adapter only after verifying the packaged flow. See [Better Auth](https://better-auth.com/docs/installation).

### Local data and offline support

Use remote data as the source of truth by default. Durable offline writes and cross-device sync are optional product features, not consequences of sharing a codebase.

Desktop SQLite and remote D1 are separate stores, even though both use SQLite semantics. Drizzle migrations for D1 do not automatically migrate the desktop store; Rust-owned SQLite migrations have their own runner and lifecycle. Do not give either client privileged direct access to D1. See [D1 with Drizzle](https://orm.drizzle.team/docs/sqlite/connect-cloudflare-d1).

If offline work is required, specify local versus remote ownership, browser storage needs, queued mutations, retry/idempotency, conflict resolution, deletion propagation, and sign-out cleanup before implementation. Test interrupted sync and reconnect behavior. Do not promise offline parity until both clients implement and pass the agreed scenarios.

## Verification and delivery

- Run project formatting, linting, Go-powered TypeScript checks, and relevant Vitest suites. Test shared business rules once and platform implementations separately, including unsupported-capability behavior.
- Use Playwright for browser flows and shared UI behavior. Run Rust checks and native tests for desktop; validate packaged applications on every supported OS. Browser tests with mocked IPC do not cover native permissions or installers.
- Build both clients in CI. Check that browser startup and shared routes work without Tauri, native-only actions have defined browser behavior, server-only imports stay out of the frontend, and desktop output is static and works without a development server.
- Exercise API contracts with both clients, including an older supported desktop release. Test authentication differences and database migrations; include offline scenarios only when supported.
- Deploy web/API previews separately from desktop builds. Use separate secrets and release jobs for hosting and native signing. Never ship server or signing credentials to either client.
- Release desktop through signed installers and the selected update channel. Test upgrade/migration from the prior supported version; keep API compatibility across independent web/backend and desktop release schedules.

Follow [Tauri testing](https://v2.tauri.app/develop/tests/), [distribution](https://v2.tauri.app/distribute/), and [updater](https://v2.tauri.app/plugin/updater/) guidance for native release details.

## Project kickoff checklist

- [ ] One shared SPA; document any separate public-site or SSR requirement as an architecture exception.
- [ ] Feature parity matrix: shared, browser-only, desktop-only, and unsupported behavior.
- [ ] Single-project module boundaries, native adapters, and browser equivalents.
- [ ] API contracts, server authorization, desktop compatibility window, and breaking-change policy.
- [ ] Browser and desktop authentication flows, origins, callbacks, credential storage, and logout.
- [ ] Remote source of truth, local caching, and whether offline writes/sync are in scope.
- [ ] Independent remote/local migrations, backups, and recovery.
- [ ] Web hosting, desktop target matrix, signing, update channels, and coordinated release checks.
- [ ] Logging/error reporting, privacy, operating costs, and app-version identification across clients.

This is a documentation standard, not a scaffold. Add native capabilities and platform abstractions when the product requires them.
