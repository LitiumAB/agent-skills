# Migrating from Module Federation

Automated migration from Angular Module Federation extensions to the Web Component approach.

## Should You Migrate?

| Question | If yes... |
|---|---|
| Extension built with Angular Module Federation (`webpack.config.js` + `ModuleFederationPlugin`)? | You can migrate |
| Extension shares Angular with the host (`singleton: true` in MF config)? | You **should** migrate — host Angular upgrades force extension rebuilds |
| You want to use React, Vue, or a different Angular version? | You **must** migrate first |
| Extension is already a Custom Element? | No migration needed |

---

## Step 1 — Back Up

```bash
git checkout -b migrate/module-federation-to-web-component
```

---

## Step 2 — Dry Run

Preview every change without writing to disk:

```bash
npx @litium/platform-extension-sdk migrate --source ./my-extension --dry-run
```

Review the output — it lists each file that will be created or modified.

---

## Step 3 — Run the Migration

```bash
npx @litium/platform-extension-sdk migrate --source ./my-extension
```

### What the CLI Does Automatically

| Action | Before | After |
|---|---|---|
| Archives webpack config | `webpack.config.js` active | `webpack.config.mf-backup.js` (inactive) |
| Creates Vite config | — | `vite.config.ts` (IIFE build) |
| Converts NgModules | `ModuleFederationPlugin` exposed modules | `@angular/elements` Custom Element registrations |
| Replaces host service calls | `NotificationActions.dispatch(...)` | `window.litiumExtension.showNotification(...)` |
| Replaces host navigation | `Router.navigate(...)` (shared) | `window.litiumExtension.navigate(...)` |
| Replaces authenticated HTTP | `HttpClient.get/post/...` | native `fetch()` (replace with `adminFetch` for authenticated endpoints) |
| Updates manifest | `frameworkType: 'angular-module'` | `frameworkType: 'web-component'` |
| Updates dependencies | `webpack`, `@angular-architects/module-federation` | `vite`, `@litium/platform-extension-sdk` |
| Configures registry | (may be missing) | `.npmrc` with `@litium:registry=https://packages.litium.com/Npm/` |
| Documents remaining steps | — | `MIGRATION_REPORT.md` |

### Before/After Examples

**Notifications:**
```typescript
// Before
import { NotificationActions } from 'litium-ui';
this.store.dispatch(NotificationActions.showSuccess({ message: 'Saved!' }));

// After
window.litiumExtension.showNotification({ message: 'Saved!', type: 'success' });
```

**Navigation:**
```typescript
// Before — shared Angular Router from the host
this.router.navigate(['/settings/extensions/my-extension/detail']);

// After
window.litiumExtension.navigate('/my-extension/detail');
```

**HTTP:**
```typescript
// Before
import { HttpClient } from '@angular/common/http';
this.http.get('/Litium/api/my-endpoint').subscribe(...);

// After (migration output — replace with adminFetch for authenticated endpoints)
import { adminFetch } from '../lib/adminFetch.js';
const res = await adminFetch('/Litium/api/my-endpoint', { method: 'GET' });
const data = await res.json();
```

---

## Step 4 — Read `MIGRATION_REPORT.md`

The CLI generates `MIGRATION_REPORT.md` in the extension root. It lists:
- All automatic changes applied
- Items marked `TODO` requiring manual attention
- `litium-ui` component usage that could not be automatically replaced

### Common Manual Items

**TranslateService:**
The CLI replaces `TranslateService.instant()` with a `// TODO` comment. Options:
- Bundle your own i18n library (e.g. `ngx-translate`, `i18next`)
- Use `window.litiumExtension.getContext().language` and maintain your own translation map

**Shared state (NgRx / Redux):**
Host-provided store state is not accessible from a Web Component. Refactor to use `adminFetch` for data fetching or bundle your own state management.

**`litium-ui` components:**
Visual components from `litium-ui` (tables, buttons, form controls) are part of the host app and cannot be used in a bundled Custom Element. Replace with your own components or a UI library (PrimeNG, Material, Radix UI, etc.).

---

## Step 5 — Install and Build

```bash
cd my-extension
npm install
npm run build
```

Fix any TypeScript errors — the most common are unresolved imports referencing host-provided Angular modules.

---

## Step 6 — Test

1. Start the dev server: `npm run dev`
2. Register in **Settings > Extensions** with `bundleUrl: 'http://localhost:3000/src/main.ts'`
3. Verify each route loads correctly
4. Click every link and button — confirm `showNotification`, `navigate`, `adminFetch` work
5. Test browser back/forward navigation

---

## Troubleshooting

| Symptom | Root cause | Fix |
|---------|-----------|-----|
| `Cannot find module 'litium-ui'` | Unreplaced `litium-ui` imports | Remove all `litium-ui` imports (flagged in `MIGRATION_REPORT.md`) |
| Blank screen, no console errors | `zone.js` imported after Angular imports | Import `zone.js` **before** any Angular import in `main.ts` |
| `customElements.define` called twice | Missing guard on hot-reload | Wrap with `if (!customElements.get('litium-ext-...'))` |
| Shared state unavailable | NgRx store was host-provided | Refactor to use `getContext()` / `adminFetch` or bundle your own state management |
