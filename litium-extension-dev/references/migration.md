# Migrating from Module Federation

Manual migration from Angular Module Federation extensions to the Web Component approach.

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

## Step 2 — Migrate Manually

### Archive the webpack config

Rename (do not delete) the existing webpack config so it is no longer active:

```bash
mv webpack.config.js webpack.config.mf-backup.js
```

### Add Vite and the SDK

```bash
npm remove webpack @angular-architects/module-federation
npm install --save-dev vite @litiumab/platform-extension-sdk
```

Create `vite.config.ts`:

```typescript
import { defineConfig } from 'vite';
import { litiumExtensionPlugin } from '@litiumab/platform-extension-sdk/vite-plugin';

export default defineConfig({
  plugins: [litiumExtensionPlugin()],
  build: { lib: { entry: 'src/index.ts', formats: ['iife'], name: 'extension' } },
});
```

### Configure the npm registry

Ensure `.npmrc` contains:

```ini
@litiumab:registry=https://registry.npmjs.org/
```

### Convert NgModules to Custom Elements

Replace `ModuleFederationPlugin` exposed modules with `@angular/elements` Custom Element registrations in `src/index.ts`:

```typescript
import { createApplication } from '@angular/platform-browser';
import { createCustomElement } from '@angular/elements';
import { MyComponent } from './app/my.component';

if (!customElements.get('litium-ext-my-extension')) {
  createApplication().then(app => {
    const el = createCustomElement(MyComponent, { injector: app.injector });
    customElements.define('litium-ext-my-extension', el);
  });
}
```

### Replace host service calls

Replace all calls to host-provided Angular services with `window.litiumExtension` equivalents. See the Before/After examples below.

### Update `extension.manifest.json`

Change `frameworkType` from `'angular-module'` to `'web-component'`.

---

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

## Step 3 — Address Common Manual Items

**TranslateService:**
The CLI replaces `TranslateService.instant()` with a `// TODO` comment. Options:
- Bundle your own i18n library (e.g. `ngx-translate`, `i18next`)
- Use `window.litiumExtension.getContext().language` and maintain your own translation map

**Shared state (NgRx / Redux):**
Host-provided store state is not accessible from a Web Component. Refactor to use `adminFetch` for data fetching or bundle your own state management.

**`litium-ui` components:**
Visual components from `litium-ui` (tables, buttons, form controls) are part of the host app and cannot be used in a bundled Custom Element. Replace with your own components or a UI library (PrimeNG, Material, Radix UI, etc.).

---

## Step 4 — Install and Build

```bash
cd my-extension
npm install
npm run build
```

Fix any TypeScript errors — the most common are unresolved imports referencing host-provided Angular modules.

---

## Step 5 — Test

1. Start the dev server: `npm run dev`
2. Register in **Settings > Extensions** with `bundleUrl: 'http://localhost:3000/src/main.ts'`
3. Verify each route loads correctly
4. Click every link and button — confirm `showNotification`, `navigate`, `adminFetch` work
5. Test browser back/forward navigation

---

## Troubleshooting

| Symptom | Root cause | Fix |
|---------|-----------|-----|
| `Cannot find module 'litium-ui'` | Unreplaced `litium-ui` imports | Remove all `litium-ui` imports |
| Blank screen, no console errors | `zone.js` imported after Angular imports | Import `zone.js` **before** any Angular import in `main.ts` |
| `customElements.define` called twice | Missing guard on hot-reload | Wrap with `if (!customElements.get('litium-ext-...'))` |
| Shared state unavailable | NgRx store was host-provided | Refactor to use `getContext()` / `adminFetch` or bundle your own state management |
