# Tauri desktop tech stack standard

Last reviewed: November 1, 2026.

Use this standard for new desktop applications. Inherit the frontend tooling, version policy, UI conventions, and conditional libraries from the [web standard](tech-stack.md), with the desktop-specific decisions below taking precedence. For a product shipping both browser and desktop applications, also follow the [shared web/desktop standard](web-desktop-tech-stack.md). Record project exceptions and their reasons.

## Required defaults

| Area | Standard |
| --- | --- |
| Desktop framework | Latest stable, compatible Tauri release; Tauri 2 at review time |
| Frontend | React, strict TypeScript, Vite+, and pnpm |
| Styling and components | Tailwind CSS, shadcn/ui, and appropriate TanStack libraries |
| Routing | TanStack Router; bundled SPA by default |
| Native implementation | Rust, Cargo, and rustup |
| JavaScript environment | Global Vite+ CLI for pinned Node.js and pnpm |
| Rust environment | Pin a supported stable toolchain in `rust-toolchain.toml`; commit `Cargo.lock` for the application |
| Native integration | Official Tauri plugins first; narrow Rust commands for application-specific operations |
| Local relational data | SQLite when needed, owned by the Rust layer |
| Remote backend | Hono when a separate API is needed; Drizzle and D1 on Cloudflare per the web standard |
| Authentication | Better Auth on the remote backend when required |

Keep Tauri's CLI, JavaScript API, Rust crates, and plugins on compatible versions. Vite+ manages the JavaScript toolchain; it does not replace rustup, Cargo, platform SDKs, or operating-system build dependencies. Follow [Tauri prerequisites](https://v2.tauri.app/start/prerequisites/) for each target.

### Frontend and build workflow

Bundle the frontend locally. Tauri serves compiled HTML, CSS, and JavaScript in an operating-system webview and does not natively run an SSR application server. Use Vite+ to build the React SPA; do not package a TanStack Start SSR server or introduce a Node.js sidecar by default. See [frontend configuration](https://v2.tauri.app/start/frontend/).

Configure Tauri's `beforeDevCommand` and `beforeBuildCommand` to invoke the project's frontend scripts, with those scripts running Vite+ development and build commands. Match `devUrl` to the configured dev-server port, use a strict port, and point `frontendDist` to the actual compiled output relative to the Tauri configuration. Follow the [Vite integration guide](https://v2.tauri.app/start/frontend/vite/) and verify routing in a packaged build.

Use the project-local Tauri CLI. From the application package, the normal entry points are:

```sh
pnpm exec tauri dev
pnpm exec tauri build
```

These run the native application workflow; `vp dev` alone only runs the frontend server. Keep frontend and native output paths separate, and confirm the Vite+ and Tauri integrations with a development launch and release build before feature work.

## Native boundaries and persistence

- Keep rendering and UI state in React. Put privileged OS access and local persistence behind Rust commands or approved plugins. Prefer command request/response for operations and events for notifications; unsubscribe event handlers when consumers unmount.
- Define typed command arguments, results, and errors. Centralize TypeScript wrappers around `invoke`; validate untrusted arguments in Rust even when Zod already validated the UI input. Keep filesystem paths, command execution, and database operations narrowly scoped.
- Use SQLite for structured local data when required. Default to Rust-owned access through SQLx with its SQLite support and versioned migrations. Use Cargo for Rust dependencies; Drizzle remains the default ORM for the remote TypeScript backend, not the Rust layer. A frontend SQL-plugin design is an explicit exception, not an assumed Drizzle-compatible driver.
- Store application data in OS-appropriate application directories. Define migrations, backup/export, corruption handling, and account deletion behavior before retaining important data.
- D1 is a remote server database, not an embedded offline store. Never bundle its administrative credentials or connect clients directly with server privileges.
- Keep long-lived credentials in an explicitly selected OS-backed credential store. Ordinary local storage, a settings file, or a local SQLite database is not a secret vault. Do not embed server secrets in JavaScript or the Rust binary.

References: [Rust commands](https://v2.tauri.app/develop/calling-rust/), [events](https://v2.tauri.app/develop/calling-frontend/), [SQLx](https://github.com/launchbadge/sqlx), and [Tauri SQL plugin](https://v2.tauri.app/plugin/sql/).

### Capabilities and authentication

Grant only the capabilities and plugin permissions required by each window. Scope filesystem and external URL access, define a production content security policy, and keep remote pages outside the privileged application surface. Review custom-command permissions explicitly; do not assume registering a command makes its access appropriately restricted. See [capabilities](https://v2.tauri.app/security/capabilities/) and [content security policy](https://v2.tauri.app/security/csp/).

When remote authentication is required, keep Better Auth on the server. Design a desktop session flow explicitly: use the system browser for OAuth, a supported callback mechanism, state validation and PKCE where applicable, credential storage, logout, and revocation. Verify origin, cookie, and redirect behavior in the packaged application; browser sessions do not automatically transfer to a webview. Evaluate desktop auth adapters for maintenance and compatibility before adopting one. See [Better Auth](https://better-auth.com/docs/installation) and [Tauri deep links](https://v2.tauri.app/plugin/deep-linking/).

## Tests, releases, and updates

- Run the inherited frontend static checks and Vitest tests. Use Playwright for browser-rendered UI behavior; mocks of Tauri APIs do not prove native behavior.
- Run Rust formatting, Clippy with warnings denied, and Cargo tests against the committed lockfile in CI. Test command validation, native errors, and local database migrations.
- Build on supported platform runners and test the actual installers/webviews. Declare supported operating-system versions and CPU architectures; include install, launch, routing, native permissions, data persistence, and uninstall expectations.
- Use Tauri's native testing/WebDriver guidance where supported. Current desktop WebDriver coverage excludes macOS; retain a packaged-app smoke-test procedure for gaps. See [Tauri testing](https://v2.tauri.app/develop/tests/).
- Use platform code signing and macOS notarization as required by the chosen distribution channel. Keep signing credentials in CI secrets and document certificate renewal. See [distribution](https://v2.tauri.app/distribute/).
- For direct distribution requiring in-app updates, use the official updater plugin with HTTPS and signed update artifacts. Updater signatures and OS code signing are separate requirements. Protect and back up signing keys; test an upgrade from a prior installed version, including database migration and failure recovery. Store distribution may require a different update path. See [updater](https://v2.tauri.app/plugin/updater/).
- Cloudflare or Vercel can host the remote services and download/update endpoints where suitable; desktop applications ship as platform installers, not web deployments.

## Project kickoff checklist

- [ ] Supported operating systems, architectures, minimum versions, and actual webview behavior.
- [ ] Local-only versus connected operation; local data ownership, backup, and optional sync requirements.
- [ ] Required native plugins, custom commands, capabilities, windows, and CSP.
- [ ] Desktop UX: menus, shortcuts, dialogs, tray, notifications, window lifecycle, accessibility, and high-DPI behavior as needed.
- [ ] Authentication callback mechanism, secure storage, session expiry, logout, and revocation if connected.
- [ ] Distribution channel, installer formats, signing/notarization, updater ownership, and release channels.
- [ ] CI platform matrix, native smoke tests, crash/error reporting, and release recovery.

These defaults cover desktop only. Mobile targets and additional sidecars require a separate project decision.
