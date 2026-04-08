---
name: litium-extension-dev
description: "Skill for building, maintaining, and deploying Litium backoffice extensions using the @litiumab/platform-extension-sdk. Use when: (1) scaffolding a new extension project (React, Vue, Angular, or Vanilla). (2) adding panels or settings pages to an existing extension. (3) working with the extension manifest (extension.manifest.json). (4) using the window.litiumExtension bridge API (navigate, fetch, showNotification, getContext, events). (5) implementing sub-path routing with MemoryRouter / createMemoryHistory / MemoryLocationStrategy. (6) building and deploying an IIFE bundle with the Litium Vite plugin. (7) writing unit tests that mock window.litiumExtension. (8) migrating an Angular Module Federation extension to a Web Component. (9) installing, enabling, or managing extensions via the Extension Management API or the Settings > Extensions UI."
---

# Litium Extension Developer

Skill for building Litium backoffice UI extensions using `@litiumab/platform-extension-sdk`. Covers scaffolding, coding, testing, deploying, and migrating extensions for all supported frameworks.

## Architecture Overview

```
┌───────────────────────────────────────────────────────────┐
│  Litium Admin (Angular 20 host)                           │
│                                                           │
│  ┌─────────────────────────────┐                         │
│  │  <litium-ext-my-widget>     │  ← Custom Element       │
│  │                             │    (your extension)     │
│  │  ┌─────────────────────┐   │                         │
│  │  │   Your framework    │   │                         │
│  │  │   (React / Vue /    │   │                         │
│  │  │    Angular / Vanilla│   │                         │
│  │  └─────────────────────┘   │                         │
│  │            │               │                         │
│  │            ▼               │                         │
│  │  window.litiumExtension     │  ← Bridge API           │
│  └─────────────────────────────┘                         │
│              │                                            │
│              ▼                                            │
│  navigate() · showNotification() · fetch() · getContext() │
│  on('routeChanged') · on('contextChanged') · off()        │
└───────────────────────────────────────────────────────────┘
```

Extensions are **Web Components (Custom Elements)** loaded as IIFE bundles. They run in complete isolation from the Angular 20 host — any front-end framework works. The host communicates with extensions exclusively through `window.litiumExtension`.

## Reference Files

Load the relevant reference when working on a task:

| Task | Reference |
|------|-----------|
| SDK CLI commands: create, add, dev, migrate | [references/cli-commands.md](references/cli-commands.md) |
| extension.manifest.json fields and targets | [references/manifest-reference.md](references/manifest-reference.md) |
| window.litiumExtension bridge API | [references/bridge-api.md](references/bridge-api.md) |
| Framework-specific patterns (React, Vue, Angular, Vanilla) | [references/framework-patterns.md](references/framework-patterns.md) |
| Sub-path routing (MemoryRouter, etc.) | [references/routing.md](references/routing.md) |
| Unit testing, mocking, CI | [references/testing.md](references/testing.md) |
| Migrating from Module Federation | [references/migration.md](references/migration.md) |
| Installing/deploying extensions | [references/deployment.md](references/deployment.md) |

## Critical Rules

These rules MUST be followed in every extension. Violating them causes runtime failures.

### Always Use the CLI — Never Create Files Manually
When scaffolding a new project, adding a panel, or adding a settings page, you **MUST** use the `@litiumab/platform-extension-sdk` CLI (see [references/cli-commands.md](references/cli-commands.md)). Do **NOT** manually create component files, manifest entries, or index imports for these operations — the CLI handles all three correctly and consistently.

Only write code manually _inside_ the generated component files (implementing the UI logic). Never manually scaffold the file structure, manifest entries, or index imports that the CLI would otherwise create.

### Custom Element Tag Naming
- All tags MUST start with `litium-ext-`
- Pattern: `litium-ext-{extension-id}` for the main element
- Panels: `litium-ext-{extension-id}-panel-{slug}`
- Must match regex: `/^litium-ext-[a-z][a-z0-9-]*$/`

### Guard `customElements.define`
Always wrap with a guard to prevent hot-reload errors:
```typescript
if (!customElements.get('litium-ext-my-extension')) {
  customElements.define('litium-ext-my-extension', MyElement);
}
```

### Cleanup Event Subscriptions
Always unsubscribe from `window.litiumExtension.on()` in your cleanup lifecycle hook (`disconnectedCallback` / `onUnmounted` / `ngOnDestroy`). Failing to do so causes memory leaks.

### Never Call `navigate()` Inside a `routeChanged` Handler
This creates an infinite loop. Only call `navigate()` in response to user actions.

### Use Memory-Based Routing
Extensions must NOT call `history.pushState` directly. Use MemoryRouter (React), createMemoryHistory (Vue), or MemoryLocationStrategy (Angular). The host owns the browser history.

### NPM Registry Configuration
The `@litiumab/platform-extension-sdk` package requires the Litium private npm registry. The `.npmrc` file must contain:
```ini
@litiumab:registry=https://registry.npmjs.org/
```

### Use the `admin-fetch` Subpath for `createAdminFetch`
Always import `createAdminFetch` from `@litiumab/platform-extension-sdk/admin-fetch`, **not** from the package root. The root entry re-exports CLI helpers (`scaffold`, `migrate`) that import `fs-extra` at the top level. Importing from the root causes Vite to pull `fs-extra` → `graceful-fs` into the browser bundle, producing Node.js compatibility errors at runtime.

```typescript
// ✓ Correct
import { createAdminFetch } from '@litiumab/platform-extension-sdk/admin-fetch';

// ✗ Wrong — pulls fs-extra into the browser bundle
import { createAdminFetch } from '@litiumab/platform-extension-sdk';
```

## Quick Workflows

### Creating a New Extension

> **Always scaffold with the CLI.** Do not create project files manually.


1. Read [references/cli-commands.md](references/cli-commands.md)
2. Ask the user which framework they want: `react`, `vue`, `angular`, or `vanilla`
3. Run: `npx @litiumab/platform-extension-sdk create <name> --framework <framework>`
4. `cd <name> && npm install`
5. Read [references/framework-patterns.md](references/framework-patterns.md) for the chosen framework
6. Start dev server: `npm run dev`
7. Register in Litium via Settings > Extensions (see [references/deployment.md](references/deployment.md))

### Adding a Panel / Settings Page

> **Always use the CLI.** Do not manually create component files, manifest entries, or index imports — the CLI handles all three correctly.


1. Read [references/cli-commands.md](references/cli-commands.md)
2. Run `npx @litiumab/platform-extension-sdk add` from the extension project root (interactive) or use:
   - `npx @litiumab/platform-extension-sdk add panel`
   - `npx @litiumab/platform-extension-sdk add settings-page`
3. The CLI generates the component file, updates the manifest, and injects the import
4. For settings pages: add the route in the framework router (see [references/routing.md](references/routing.md))

### Building and Deploying

1. Read [references/deployment.md](references/deployment.md)
2. `npm run build` — produces `dist/extension.js`
3. Upload `extension.js` to a CDN or hosting server (HTTPS required for production)
4. Update `bundleUrl` in `extension.manifest.json` to the production URL
5. Install via Settings > Extensions in the backoffice, or POST to the Extension Management API

### Migrating from Module Federation

1. Read [references/migration.md](references/migration.md)
2. Run: `npx @litiumab/platform-extension-sdk migrate --source ./my-extension --dry-run` (preview)
3. Run: `npx @litiumab/platform-extension-sdk migrate --source ./my-extension` (apply)
4. Read `MIGRATION_REPORT.md` for manual TODO items
5. `npm install && npm run build` — fix TypeScript errors
6. Test all routes and bridge API calls

### Writing Unit Tests

1. Read [references/testing.md](references/testing.md)
2. Mock `window.litiumExtension` in `beforeEach` — the reference has the full mock template
3. Test component rendering, navigation calls, and event subscriptions
4. Run: `npm test`

## Common Mistakes

| Symptom | Root cause | Fix |
|---------|-----------|-----|
| 403 Forbidden on API calls | Using native `fetch` instead of `window.litiumExtension.fetch()` | Replace with `window.litiumExtension.fetch()` |
| Extension blank / not rendering | Custom element tag mismatch between manifest and code | Ensure tag in `customElements.define()` matches `customElementTag` in manifest |
| Back button doesn't work | Using framework router directly instead of bridge API | Call `window.litiumExtension.navigate()` for cross-boundary navigation |
| Infinite re-renders | Calling `navigate()` inside a `routeChanged` handler | Only call `navigate()` in response to user actions |
| Hot-reload error: element already defined | Missing `customElements.get()` guard | Wrap `customElements.define()` with the guard pattern |
| `npm ERR! 404 Not Found` for SDK | Missing `.npmrc` registry config | Add `@litiumab:registry=https://registry.npmjs.org/` to `.npmrc` |
| Vite bundle error: `fs`, `graceful-fs`, or Node built-ins in browser | `createAdminFetch` imported from package root, pulling `fs-extra` | Change import to `@litiumab/platform-extension-sdk/admin-fetch` |
| Extension not in Settings menu | Extension not installed or disabled | Install via Settings > Extensions or POST to API; enable if disabled |
| Deep links load root page instead | Not reading `sub-path` attribute on first render | Read `getAttribute('sub-path')` in `connectedCallback` |
| Stale context after channel switch | Caching `getContext()` result forever | Subscribe to `contextChanged` event |
| Memory leak warnings on unmount | Not unsubscribing from events | Call the unsubscribe function in `disconnectedCallback` / cleanup hook |
