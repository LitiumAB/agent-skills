# CLI Commands

Complete reference for the `@litiumab/platform-extension-sdk` CLI.

## Prerequisites

Configure the Litium private npm registry before running any CLI command:

```ini
# .npmrc (project root or ~/.npmrc)
@litiumab:registry=https://registry.npmjs.org/
```

Verify access:
```bash
npm show @litiumab/platform-extension-sdk
```

---

## `create` — Scaffold a New Extension

```bash
npx @litiumab/platform-extension-sdk create <name> --framework <react|vue|angular|vanilla>
```

| Flag | Required | Description |
|------|----------|-------------|
| `<name>` | Yes | Extension name (kebab-case) |
| `--framework` | Yes | `react`, `vue`, `angular`, or `vanilla` |

### Generated project structure

| File / folder | Purpose |
|---|---|
| `src/index.ts` (or `.tsx`) | Custom Element entry point |
| `src/App.tsx` / `App.vue` / `app.module.ts` | Root application component |
| `src/pages/` | Sample page components |
| `vite.config.ts` | IIFE build with Litium Vite plugin |
| `extension.manifest.json` | Deployment descriptor |
| `.npmrc` | Pre-configured with `@litium:registry` |
| `.dev.config.json` | Dev server port (default `3000`) |
| `vitest.config.ts` or `jest.config.ts` | Test configuration |

### After scaffolding

```bash
cd <name>
npm install
npm run dev    # starts Vite dev server
```

---

## `add` — Add Extension Points

Run from the extension project root (directory containing both `package.json` and `extension.manifest.json`).

```bash
npx @litiumab/platform-extension-sdk add              # interactive prompt
npx @litiumab/platform-extension-sdk add panel        # add an area panel
npx @litiumab/platform-extension-sdk add settings-page # add a settings page
```

### `add panel`

Generates a web component panel for one of the five backoffice areas: **Customers**, **Products**, **Sales**, **Media**, **Websites**.

**Prompts:**
| Prompt | Example |
|---|---|
| Panel display name | `Pricing Rules` |
| Admin area | `products` |

**What gets generated:**
- `src/panels/<slug>.tsx` (or `.vue` / `.ts`) — panel web component
- `extension.manifest.json` — updated with `{area}.menu.item` target
- `src/index.ts` — import injected automatically

**Generated manifest entry:**
```json
{
  "target": "products.menu.item",
  "name": "Pricing Rules",
  "component": "litium-ext-my-extension-panel-pricing-rules"
}
```

Custom element tag pattern: `{customElementTag}-panel-{slug}`.

### `add settings-page`

Adds a new page to the Settings sidebar.

**Prompts:**
| Prompt | Default | Description |
|---|---|---|
| Display name | — | Label in the Settings sidebar |
| Settings group ref ID | `system.settings` | The settings group |
| URL path | `/Litium/UI/settings/extensions/{id}` | Full path |

**What gets generated:**
- `src/settings/<slug>.tsx` (or equivalent) — page component
- `extension.manifest.json` — updated with `settings.menu.item` target
- `src/index.ts` — import injected automatically

**Generated manifest entry:**
```json
{
  "target": "settings.menu.item",
  "name": "Integration Config",
  "ref_id": "system.settings",
  "url": "/Litium/UI/settings/extensions/my-extension/integration-config"
}
```

**Remaining manual step:** Add the route in your framework router:

| Framework | Where | Code |
|---|---|---|
| React | `src/App.tsx` | `<Route path="/integration-config" element={<IntegrationConfigPage />} />` |
| Vue | `src/router.ts` | `{ path: '/integration-config', component: () => import('./settings/integration-config.vue') }` |
| Angular | Routes array | `{ path: 'integration-config', component: IntegrationConfigPageComponent }` |
| Vanilla | Routing logic | `router.register('/integration-config', renderIntegrationConfigPage)` |

---

## `dev` — Local Development Server

```bash
npm run dev
```

Starts a Vite dev server (port from `.dev.config.json`, default `3000`) that serves:
- `http://localhost:<port>/extension.manifest.json` — the manifest
- `http://localhost:<port>/src/index.ts` (or `.tsx`) — the HMR-enabled bundle

During local development, `bundleUrl` in the manifest should point at the dev server URL (e.g. `http://localhost:3000/src/index.ts`).

---

## `build` — Production Build

```bash
npm run build
```

Produces `dist/extension.js` — a single IIFE bundle. The Litium Vite plugin (`@litiumab/platform-extension-sdk/vite-plugin`) handles:
- Validating `customElementTag` starts with `litium-ext-`
- Building as a single-file IIFE
- Generating `dist/extension.manifest.json` with production `bundleUrl`

---

## `migrate` — Migrate from Module Federation

```bash
npx @litiumab/platform-extension-sdk migrate --source ./my-mf-extension --dry-run  # preview
npx @litiumab/platform-extension-sdk migrate --source ./my-mf-extension            # apply
```

| Flag | Required | Description |
|------|----------|-------------|
| `--source` | Yes | Path to the existing Module Federation extension |
| `--dry-run` | No | Preview changes without writing to disk |

See [migration.md](migration.md) for the full migration workflow.
