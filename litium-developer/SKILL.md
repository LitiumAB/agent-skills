---
name: litium-developer
description: "Help Litium partner developers build, configure, and deploy e-commerce websites and backoffice UI extensions using the Litium platform. Use for React Accelerator (Next.js, GraphQL, blocks, cart, checkout, wishlist), MVC Accelerator (.NET Razor views, controllers, view models, client builds), local environment setup (empty project, MVC Accelerator, React Accelerator), deploy to Litium Cloud, custom field types, field templates, multi-field, data modelling, Admin Web API, Storefront GraphQL API, Connect ERP/Payment/Shipment API, Extension Management API, and searching forum.litium.com for community solutions. Also covers building backoffice UI extensions with @litiumab/platform-extension-sdk: scaffolding extensions in React, Vue, Angular, or Vanilla; extension manifest (extension.manifest.json); window.litiumExtension bridge API; sub-path routing with MemoryRouter; building IIFE bundles with the Litium Vite plugin; unit testing; installing and deploying extensions. Triggers on: build Litium site, create Litium project, add product fields, create block, set up locally, configure accelerator, Litium API, wishlist, cart mutation, placeOrder, custom field, create extension, backoffice extension, platform-extension-sdk, UI extension, add panel, settings page extension."
---

# Litium Developer

## Overview

Partner developers building on Litium platform use this skill for implementation guidance. This skill delegates to the `litium-cloud-cli` sub-skill for cloud deployment. For all other Litium topics — including React Accelerator, MVC, data modelling, and APIs — it loads reference files on demand.

Always read `references/partner-guidance.md` first before helping modify Litium platform files.

## Delegation

For this area, delegate entirely to the named skill — do not duplicate its content here:

| Topic | Skill to use |
|-------|-------------|
| Deploy to Litium Cloud, create/manage environments, artifacts, CI/CD, YAML apply manifests, service principals | `litium-cloud-cli` |

## Reference map

Read these files **only** when the user's task requires that area:

| Reference file | Load when the user asks about |
|----------------|-------------------------------|
| `references/partner-guidance.md` | what partners can/cannot modify, correct extension patterns, internal API disclaimer, protected projects, NuGet feeds, Elasticsearch index separation |
| `references/setup-empty-project.md` | set up bare Litium, empty project, install bare platform, download Litium locally, `litemptyweb` template |
| `references/setup-mvc-accelerator.md` | install MVC Accelerator locally, `litmvcacc` template, yarn build, deploy accelerator in back office |
| `references/setup-react-accelerator.md` | install React Accelerator locally, `litreactacc` template, storefront proxy, `.env.local`, headless accelerator |
| `references/setup-troubleshooting.md` | setup error, local environment issue, install fails, SQL connection, certificate, port conflict, package restore |
| `references/mvc-accelerator.md` | MVC development, .NET controller, Razor view, view model, webpack client build, MVC + React component, Elasticsearch MVC |
| `references/data-modelling.md` | custom field types, create field type, field template, folder template, product template, multi-field, field not in template, `FieldTypeMetadataBase` |
| `references/cart-and-wishlist.md` | cart, checkout, order, wishlist, `addVariantToCart`, `createCheckoutSession`, `placeOrder`, `CartContext`, `BuyButton`, payment widget |
| `references/apis.md` | Admin Web API, Storefront GraphQL API, Connect ERP API, Payment API, Shipment API, Extension Management API, OpenAPI spec |
| `references/forum-api.md` | search forum.litium.com, community solution, Discourse API, find forum post |
| `references/react-accelerator/overview.md` | React Accelerator development, Next.js architecture, server vs. client components, GraphQL patterns, component conventions, testing |
| `references/react-accelerator/template-routing.md` | create page/product/category/block template, proxy routing, `[[...slug]]` catch-all, template naming conventions |
| `references/react-accelerator/code-patterns.md` | React Accelerator code patterns, `queryServer`, `mutateClient`, service layers, fragments, Apollo Client usage |
| `references/react-accelerator/project-structure.md` | React Accelerator directory layout, `app/`, `components/`, `services/`, `operations/` folders |
| `references/react-accelerator/graphql-sdl.md` | fetch Storefront GraphQL SDL, schema download, `RUNTIME_LITIUM_SERVER_URL`, available operations |
| `references/extension/cli-commands.md` | scaffold new extension, `create` command, `add` command, `dev` command, `@litiumab/platform-extension-sdk` CLI |
| `references/extension/manifest-reference.md` | `extension.manifest.json` fields, targets, `common.web-component`, `admin.menu-item`, `admin.settings-page`, `api.proxy`, `customElementTag`, `bundleUrl` |
| `references/extension/bridge-api.md` | `window.litiumExtension`, `navigate()`, `replaceUrl()`, `showNotification()`, `getContext()`, `on()`, `off()`, `routeChanged`, `contextChanged` |
| `references/extension/framework-patterns.md` | Custom Element entry point, React extension, Vue extension, Angular extension, Vanilla extension, panel component, Admin Web API requests |
| `references/extension/routing.md` | sub-path routing, MemoryRouter, `createMemoryHistory`, MemoryLocationStrategy, `sub-path` attribute, `routeChanged`, deep link |
| `references/extension/testing.md` | unit testing extensions, mock `window.litiumExtension`, Vitest, Jest, `@testing-library/react`, `@vue/test-utils` |
| `references/extension/deployment.md` | install extension, enable extension, Extension Management API, `bundleUrl`, HTTPS, CDN, Settings > Extensions, IIFE bundle |

## Documentation search priority

1. **Primary**: https://docs.litium.dev/ — always try first for official docs.
2. **Community**: https://forum.litium.com/ — for real-world solutions; load `references/forum-api.md` to search programmatically.

## Quick-start routing

**Step 0 (always):** Read `references/partner-guidance.md` first to understand what partners can/cannot modify.

- **"How do I work on the React storefront?"** → read `references/react-accelerator/overview.md` first, then load specific references as needed (template-routing, code-patterns, etc.).
- **"How do I deploy/manage Litium Cloud environments?"** → delegate to `litium-cloud-cli`.
- **"How do I set up Litium locally?"** → load the matching setup reference (empty / MVC / React).
- **"How do I add cart or wishlist functionality?"** → load `references/cart-and-wishlist.md`.
- **"How do I create a custom field type or field template?"** → load `references/data-modelling.md`.
- **"How do I call the Admin Web API or a Connect API?"** → load `references/apis.md`.
- **"How do I build a backoffice UI extension?"** → read `references/extension/cli-commands.md` first, then load specific references as needed (framework-patterns, bridge-api, routing, etc.).
- **"How do I add a panel or settings page to an extension?"** → load `references/extension/cli-commands.md` (always use the CLI — never create files manually).
- **"How do I navigate or show notifications in an extension?"** → load `references/extension/bridge-api.md`.

## Backoffice UI Extension Architecture

Extensions are **Web Components (Custom Elements)** loaded as IIFE bundles. They run in complete isolation from the Angular 20 admin host — React, Vue, Angular, or Vanilla JavaScript all work. The host communicates with extensions exclusively through `window.litiumExtension`.

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
│  navigate() · showNotification() · getContext()           │
│  on('routeChanged') · on('contextChanged') · off()        │
└───────────────────────────────────────────────────────────┘
```

## Extension Critical Rules

These rules MUST be followed in every extension. Violating them causes runtime failures.

### Always Use the CLI — Never Create Files Manually
When scaffolding a new project, adding a panel, or adding a settings page, **MUST** use the `@litiumab/platform-extension-sdk` CLI (see `references/extension/cli-commands.md`). Do **NOT** manually create component files, manifest entries, or index imports — the CLI handles all three correctly.

Only write code manually _inside_ the generated component files (implementing the UI logic).

### Custom Element Tag Naming
- All tags MUST start with `litium-ext-`
- Pattern: `litium-ext-{extension-id}` for the main element
- Panels: `litium-ext-{extension-id}-panel-{slug}`
- Must match regex: `/^litium-ext-[a-z][a-z0-9-]*$/`

### Guard `customElements.define`
```typescript
if (!customElements.get('litium-ext-my-extension')) {
  customElements.define('litium-ext-my-extension', MyElement);
}
```

### Cleanup Event Subscriptions
Always unsubscribe from `window.litiumExtension.on()` in `disconnectedCallback` / `onUnmounted` / `ngOnDestroy`. Failing to do so causes memory leaks.

### Never Call `navigate()` Inside a `routeChanged` Handler
Creates an infinite loop. Only call `navigate()` in response to user actions.

### Use Memory-Based Routing
Extensions must NOT call `history.pushState` directly. Use `MemoryRouter` (React), `createMemoryHistory` (Vue), or `MemoryLocationStrategy` (Angular).

### NPM Registry Configuration
```ini
# .npmrc
@litiumab:registry=https://registry.npmjs.org/
```

## Extension Quick Workflows

### Creating a New Extension
1. Read `references/extension/cli-commands.md`
2. Ask the user which framework: `react`, `vue`, `angular`, or `vanilla`
3. Run: `npx @litiumab/platform-extension-sdk create <name> --framework <framework>`
4. `cd <name> && npm install`
5. Read `references/extension/framework-patterns.md` for the chosen framework
6. Start dev server: `npm run dev`
7. Register in Litium via Settings > Extensions (see `references/extension/deployment.md`)

### Adding a Panel / Settings Page
1. Read `references/extension/cli-commands.md`
2. From the extension project root, run `npx @litiumab/platform-extension-sdk add panel` or `add settings-page`
3. The CLI generates the component file, updates the manifest, and injects the import
4. For settings pages: add the route in the framework router (see `references/extension/routing.md`)

### Building and Deploying
1. Read `references/extension/deployment.md`
2. `npm run build` — produces `dist/extension.js`
3. Upload to a CDN or hosting server (HTTPS required for production)
4. Update `bundleUrl` in `extension.manifest.json`
5. Install via Settings > Extensions or POST to the Extension Management API

## Extension Common Mistakes

| Symptom | Root cause | Fix |
|---------|-----------|-----|
| 403 Forbidden on admin API calls | Calling `/Litium/api/admin/` without a valid bearer token | Request an OAuth token with `VITE_LITIUM_CLIENT_ID` and `VITE_LITIUM_CLIENT_SECRET`, then send it in the `Authorization` header |
| Extension blank / not rendering | Custom element tag mismatch between manifest and code | Ensure tag in `customElements.define()` matches `customElementTag` in manifest |
| Back button doesn't work | Using framework router instead of bridge API | Call `window.litiumExtension.navigate()` for cross-boundary navigation |
| Infinite re-renders | Calling `navigate()` inside a `routeChanged` handler | Only call `navigate()` in response to user actions |
| Hot-reload error: element already defined | Missing `customElements.get()` guard | Wrap `customElements.define()` with the guard pattern |
| `npm ERR! 404 Not Found` for SDK | Missing `.npmrc` registry config | Add `@litiumab:registry=https://registry.npmjs.org/` to `.npmrc` |
| Deep links load root page instead | Not reading `sub-path` attribute on first render | Read `getAttribute('sub-path')` in `connectedCallback` |
| Stale context after channel switch | Caching `getContext()` result forever | Subscribe to `contextChanged` event |
| Memory leak warnings on unmount | Not unsubscribing from events | Call the unsubscribe function in `disconnectedCallback` / cleanup hook |
