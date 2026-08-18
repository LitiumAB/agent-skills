# Framework Patterns

Per-framework Custom Element entry point, routing setup, and bridge API usage for all four supported frameworks.

---

## React

### Custom Element Entry Point (`src/index.ts`)

Use `.ts` (not `.tsx`) to avoid `@vitejs/plugin-react` Fast Refresh conflicts with non-component classes. Use `createElement` instead of JSX.

```typescript
import './preamble.js'; // must be first
import './index.css';
import { StrictMode, createElement } from 'react';
import { createRoot } from 'react-dom/client';
import { App } from './App.js';

class MyExtensionElement extends HTMLElement {
  private reactRoot?: ReturnType<typeof createRoot>;
  private currentSubPath = '/';
  private unsubscribeRouteChanged?: () => void;

  static get observedAttributes() {
    return ['sub-path'];
  }

  connectedCallback() {
    this.currentSubPath = this.getAttribute('sub-path') ?? '/';
    this.reactRoot = createRoot(this);
    this.unsubscribeRouteChanged = window.litiumExtension?.on(
      'routeChanged',
      ({ path }: { path: string }) => {
        this.currentSubPath = path;
        this.renderApp();
      },
    );
    this.renderApp();
  }

  attributeChangedCallback(_name: string, _old: string | null, newVal: string | null) {
    const path = newVal ?? '/';
    if (path !== this.currentSubPath) {
      this.currentSubPath = path;
      this.renderApp();
    }
  }

  disconnectedCallback() {
    this.unsubscribeRouteChanged?.();
    this.reactRoot?.unmount();
    this.reactRoot = undefined;
  }

  private renderApp() {
    this.reactRoot?.render(
      createElement(StrictMode, null, createElement(App, { subPath: this.currentSubPath })),
    );
  }
}

if (!customElements.get('litium-ext-my-extension')) {
  customElements.define('litium-ext-my-extension', MyExtensionElement);
}
```

### Root Component (`src/App.tsx`)

Uses `MemoryRouter` with `key={subPath}` to force fresh mounts on deep-link changes:

```tsx
import { MemoryRouter, Routes, Route, Navigate } from 'react-router';
import { PageA } from './pages/PageA.js';
import { PageB } from './pages/PageB.js';

interface AppProps { subPath: string; }

export function App({ subPath }: AppProps) {
  return (
    <MemoryRouter initialEntries={[subPath]} key={subPath}>
      <Routes>
        <Route path="/pagea" element={<PageA />} />
        <Route path="/pageb" element={<PageB />} />
        <Route path="*" element={<Navigate to="/pagea" replace />} />
      </Routes>
    </MemoryRouter>
  );
}
```

### Navigation from Page Components

Call `window.litiumExtension.navigate()` — do NOT use `useNavigate` from react-router for cross-boundary navigation:

```tsx
export function PageA() {
  const handleClick = () => {
    window.litiumExtension.navigate('/my-extension/pageb');
  };
  return <button onClick={handleClick}>Go to Page B</button>;
}
```

### Bridge API Usage in React

```tsx
import type { LitiumExtensionAPI } from '@litiumab/platform-extension-sdk';
declare global { interface Window { litiumExtension: LitiumExtensionAPI; } }

export function PageA() {
  const context = window.litiumExtension.getContext();

  const handleSave = async () => {
    const res = await fetch('/Litium/app/api/my-endpoint', {
      method: 'POST',
      body: JSON.stringify({ foo: 'bar' }),
    });
    if (res.ok) {
      window.litiumExtension.showNotification({ message: 'Saved!', type: 'success' });
    }
  };

  return (
    <div>
      <p>Language: {context.language}</p>
      <button onClick={handleSave}>Save</button>
    </div>
  );
}
```

### React Panel Component

Panels are simpler — no sub-path routing:

```tsx
import { createRoot } from 'react-dom/client';

function PricingRulesPanel() {
  return <div className="panel"><h2>Pricing Rules</h2></div>;
}

class PricingRulesPanelElement extends HTMLElement {
  private _root: ReturnType<typeof createRoot> | null = null;

  connectedCallback(): void {
    this._root = createRoot(this);
    this._root.render(<PricingRulesPanel />);
  }

  disconnectedCallback(): void {
    this._root?.unmount();
    this._root = null;
  }
}

customElements.define('litium-ext-my-extension-panel-pricing-rules', PricingRulesPanelElement);
```

### React Field Type Editor

Two files — React component + Custom Element wrapper (keeps Vite Fast Refresh working):

```tsx
// src/field-types/product-rating/editor.tsx
import React from 'react';

declare global {
  namespace JSX {
    interface IntrinsicElements {
      'litium-field-editor': React.HTMLAttributes<HTMLElement> & {
        label?: string; tooltip?: string; readonly?: string; errors?: string;
      };
    }
  }
}

export interface ProductRatingEditorProps {
  value: number | null;
  label?: string | null;
  readonly?: boolean;
  errors?: string[];
  onChange: (value: number) => void;
}

export function ProductRatingEditor({ value, label, readonly, errors, onChange }: ProductRatingEditorProps) {
  return (
    <litium-field-editor
      label={label ?? ''}
      {...(readonly ? { readonly: '' } : {})}
      {...(errors?.length ? { errors: JSON.stringify(errors) } : {})}
    >
      <input type="number" value={value ?? ''} onChange={(e) => onChange(Number(e.target.value))} />
      <span slot="preview">{String(value ?? '')}</span>
    </litium-field-editor>
  );
}
```

### Vite Config (React)

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { litiumExtension } from '@litiumab/platform-extension-sdk/vite-plugin';

export default defineConfig({
  plugins: [
    react(),
    litiumExtension({ customElementTag: 'litium-ext-my-extension' }),
  ],
  build: {
    lib: {
      entry: 'src/index.ts',
      formats: ['iife'],
      name: 'MyExtension',
      fileName: () => 'extension.js',
    },
    rollupOptions: { output: { inlineDynamicImports: true } },
  },
});
```

---

## Vue

### Custom Element Entry Point (`src/index.ts`)

Vue's `defineCustomElement` compiles an SFC into a Web Component with shadow DOM:

```typescript
import { defineCustomElement } from 'vue';
import type { LitiumExtensionAPI } from '@litiumab/platform-extension-sdk';
import App from './App.vue';

declare global { interface Window { litiumExtension: LitiumExtensionAPI; } }

const MyExtensionElement = defineCustomElement(App);

if (!customElements.get('litium-ext-my-extension')) {
  customElements.define('litium-ext-my-extension', MyExtensionElement);
}
```

### Root Component (`src/App.vue`)

Creates a per-instance `vue-router` with `createMemoryHistory` and syncs with the host:

```vue
<template>
  <RouterView />
</template>

<script setup lang="ts">
import { createRouter, createMemoryHistory, RouterView } from 'vue-router';
import { getCurrentInstance, watch, onUnmounted } from 'vue';
import { syncRouterFromHost } from '@litiumab/platform-extension-sdk/vue';
import PageA from './pages/PageA.vue';
import PageB from './pages/PageB.vue';

const props = defineProps<{ subPath?: string }>();

const router = createRouter({
  history: createMemoryHistory(props.subPath ?? '/'),
  routes: [
    { path: '/pagea', component: PageA },
    { path: '/pageb', component: PageB },
    { path: '/', redirect: '/pagea' },
    { path: '/:pathMatch(.*)*', redirect: '/pagea' },
  ],
});

const instance = getCurrentInstance();
instance?.appContext.app.use(router);

if (props.subPath && props.subPath !== '/') {
  syncRouterFromHost(router, props.subPath);
}

watch(() => props.subPath, (newPath) => {
  if (newPath) syncRouterFromHost(router, newPath);
});

const offRouteChanged = window.litiumExtension?.on<{ path: string }>(
  'routeChanged',
  (data) => { syncRouterFromHost(router, data.path); },
);

onUnmounted(() => offRouteChanged?.());
</script>
```

### Navigation from Page Components (Vue)

Use `createNavigate` from `@litiumab/platform-extension-sdk/vue`:

```typescript
import { createNavigate } from '@litiumab/platform-extension-sdk/vue';
import { useRouter } from 'vue-router';

const router = useRouter();
const navigate = createNavigate(router);

const goToPageB = () => navigate('/my-extension/pageb');
```

Do NOT call `router.push()` alone — it updates Vue state but not the host browser URL.

### Vite Config (Vue)

```typescript
import { defineConfig } from 'vite';
import vue from '@vitejs/plugin-vue';
import { litiumExtension } from '@litiumab/platform-extension-sdk/vite-plugin';

export default defineConfig({
  plugins: [
    vue(),
    litiumExtension({ customElementTag: 'litium-ext-my-extension' }),
  ],
  build: {
    lib: {
      entry: 'src/index.ts',
      formats: ['iife'],
      name: 'MyExtension',
      fileName: () => 'extension.js',
    },
    rollupOptions: { output: { inlineDynamicImports: true } },
  },
});
```

---

## Angular

### Entry Point (`src/main.ts`)

Creates a completely separate Angular platform — no dependency conflicts with the host:

```typescript
import 'zone.js';
import { platformBrowserDynamic } from '@angular/platform-browser-dynamic';
import { AppModule } from './app/app.module';
import type { LitiumExtensionAPI } from '@litiumab/platform-extension-sdk';

declare global { interface Window { litiumExtension: LitiumExtensionAPI; } }

platformBrowserDynamic()
  .bootstrapModule(AppModule)
  .catch((err: unknown) => console.error('[my-extension] Bootstrap error:', err));
```

### Custom Element Registration (`src/app/app.module.ts`)

```typescript
import { ApplicationRef, DoBootstrap, Injector, NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { RouterModule } from '@angular/router';
import { LocationStrategy } from '@angular/common';
import { createCustomElement } from '@angular/elements';
import { AppComponent } from './app.component';
import { routes } from './app.routes';
import { MemoryLocationStrategy } from './memory-location-strategy';

@NgModule({
  declarations: [AppComponent],
  imports: [
    BrowserModule,
    RouterModule.forRoot(routes, { initialNavigation: 'disabled' }),
  ],
  providers: [
    { provide: LocationStrategy, useClass: MemoryLocationStrategy },
  ],
  bootstrap: [],
})
export class AppModule implements DoBootstrap {
  constructor(private readonly injector: Injector) {
    const MyExtensionElement = createCustomElement(AppComponent, {
      injector: this.injector,
    });
    if (!customElements.get('litium-ext-my-extension')) {
      customElements.define('litium-ext-my-extension', MyExtensionElement);
    }
  }
  ngDoBootstrap(_appRef: ApplicationRef): void {}
}
```

### MemoryLocationStrategy

Generated in `src/app/memory-location-strategy.ts`. Implements `LocationStrategy` without calling `history.pushState`. Instead:
- Intercepts `Router.navigate()` calls → calls `window.litiumExtension.navigate()`
- Listens for `routeChanged` event → calls `Router.navigateByUrl()` back

This means standard Angular navigation APIs (`routerLink`, `Router.navigate()`, route guards) all work transparently.

### Navigation in Angular Components

```typescript
import { Router } from '@angular/router';

@Component({ /* ... */ })
export class PageAComponent {
  constructor(private router: Router) {}

  goToPageB() {
    // MemoryLocationStrategy intercepts this → calls window.litiumExtension.navigate()
    this.router.navigate(['/pageb']);
  }

  async save() {
    const res = await fetch('/Litium/api/my-endpoint', {
      method: 'POST',
    });
    if (res.ok) {
      window.litiumExtension.showNotification({ message: 'Saved!', type: 'success' });
    }
  }
}
```

### Version Isolation

Any Angular version works. The extension's `platformBrowserDynamic()` creates a new Angular platform with its own DI tree, zone.js instance, and change detection — completely separate from the host's Angular 20 platform.

### Vite Config (Angular)

```typescript
import { defineConfig } from 'vite';
import angular from '@analogjs/vite-plugin-angular';
import { litiumExtension } from '@litiumab/platform-extension-sdk/vite-plugin';

export default defineConfig({
  plugins: [
    angular(),
    litiumExtension({ customElementTag: 'litium-ext-my-extension' }),
  ],
  build: {
    lib: {
      entry: 'src/main.ts',
      formats: ['iife'],
      name: 'MyExtension',
      fileName: () => 'extension.js',
    },
    rollupOptions: { output: { inlineDynamicImports: true } },
  },
});
```

---

## Vanilla (No Framework)

### Custom Element Entry Point (`src/index.ts`)

```typescript
class MyExtensionElement extends HTMLElement {
  #currentPath = '/';
  #unsubscribe?: () => void;

  static get observedAttributes() {
    return ['sub-path'];
  }

  connectedCallback() {
    this.#currentPath = this.getAttribute('sub-path') ?? '/';
    this.#render();
    this.#unsubscribe = window.litiumExtension?.on('routeChanged', ({ path }: { path: string }) => {
      this.#currentPath = path;
      this.#render();
    });
  }

  attributeChangedCallback(_name: string, _old: string | null, newVal: string | null) {
    const path = newVal ?? '/';
    if (path !== this.#currentPath) {
      this.#currentPath = path;
      this.#render();
    }
  }

  disconnectedCallback() {
    this.#unsubscribe?.();
  }

  #render() {
    // Route manually based on this.#currentPath
    if (this.#currentPath.startsWith('/detail/')) {
      const id = this.#currentPath.split('/')[2];
      this.innerHTML = `<h1>Detail: ${id}</h1>`;
    } else {
      this.innerHTML = `<h1>Home</h1><button id="nav">Go to detail</button>`;
      this.querySelector('#nav')?.addEventListener('click', () => {
        window.litiumExtension.navigate('/my-extension/detail/123');
      });
    }
  }
}

if (!customElements.get('litium-ext-my-extension')) {
  customElements.define('litium-ext-my-extension', MyExtensionElement);
}
```

---

## Admin Web API

Endpoints under `/Litium/api/admin/` require an OAuth 2.0 bearer token. Read the service account credentials from the Vite environment variables, request a token from `/Litium/OAuth/Token`, then include the token in the Admin Web API request.

### Environment variables

```typescript
/// <reference types="vite/client" />
const clientId = import.meta.env.VITE_LITIUM_CLIENT_ID;
const clientSecret = import.meta.env.VITE_LITIUM_CLIENT_SECRET;
```

Add to `.env`:

```ini
VITE_LITIUM_CLIENT_ID=your-service-account-id
VITE_LITIUM_CLIENT_SECRET=your-service-account-secret
```

### Token and Admin Web API request

```typescript
async function getAdminAccessToken(): Promise<string> {
  const tokenResponse = await fetch('/Litium/OAuth/Token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      grant_type: 'client_credentials',
      client_id: clientId,
      client_secret: clientSecret,
    }),
  });

  if (!tokenResponse.ok) {
    throw new Error(`Token request failed: ${tokenResponse.status}`);
  }

  const token = await tokenResponse.json() as { access_token: string };
  return token.access_token;
}

const accessToken = await getAdminAccessToken();
const res = await fetch('/Litium/api/admin/sales/salesOrders/search', {
  method: 'POST',
  headers: {
    Authorization: `Bearer ${accessToken}`,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({ take: 20, skip: 0 }),
});

const data = await res.json();
```
