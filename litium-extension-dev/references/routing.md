# Extension Routing

How sub-path routing works in Litium extensions — covering all frameworks.

## How the Host Delegates Sub-Paths

The Litium admin host registers your extension against a URL prefix (e.g. `/Litium/UI/settings/extensions/my-extension`). It uses a wildcard route, so any URL under that prefix is forwarded to your extension:

```
Host URL: /Litium/UI/settings/extensions/my-extension/orders/123
                                          ^─── prefix ────^│
                                                           sub-path: /orders/123
```

The host passes the sub-path via the `sub-path` HTML attribute and fires `routeChanged` when it changes.

## The `sub-path` Attribute

| Situation | When it's set |
|---|---|
| First render | Set to the URL path at mount time (enables deep links and page refresh) |
| Browser back/forward | Attribute updated AND `routeChanged` event fired |
| `window.litiumExtension.navigate()` | Attribute updated AND `routeChanged` event fired |

Your Custom Element must:
1. Declare `sub-path` as an observed attribute
2. Read it in `connectedCallback` for initial render
3. React to `attributeChangedCallback` for subsequent changes

## Navigation Flow

### User Action → Extension → Host

```
User clicks button → window.litiumExtension.navigate('/my-ext/orders/123')
                     → Host pushes browser history entry
                     → Host fires 'routeChanged' event  { path: '/orders/123' }
                     → Your extension re-renders at new path
```

### Browser Back/Forward → Host → Extension

```
User clicks Back → Host pops history entry
                 → Host updates sub-path attribute on Custom Element
                 → Host fires 'routeChanged' event  { path: '/orders' }
                 → Your extension re-renders at previous path
```

---

## Per-Framework Setup

### React — MemoryRouter

```tsx
// src/App.tsx
import { MemoryRouter, Routes, Route, Navigate } from 'react-router';

interface AppProps { subPath: string; }

export function App({ subPath }: AppProps) {
  return (
    // key={subPath} forces a fresh MemoryRouter when the host delivers a new deep-link
    <MemoryRouter initialEntries={[subPath]} key={subPath}>
      <Routes>
        <Route path="/orders" element={<OrderListPage />} />
        <Route path="/orders/:id" element={<OrderDetailPage />} />
        <Route path="*" element={<Navigate to="/orders" replace />} />
      </Routes>
    </MemoryRouter>
  );
}
```

Navigate from page components:
```tsx
// Do NOT use useNavigate from react-router for cross-boundary navigation
const handleViewDetail = (id: string) => {
  window.litiumExtension.navigate(`/my-extension/orders/${id}`);
};
```

### Vue — createMemoryHistory + syncRouterFromHost

```vue
<script setup lang="ts">
import { createRouter, createMemoryHistory } from 'vue-router';
import { syncRouterFromHost, createNavigate } from '@litium/platform-extension-sdk/vue';

const props = defineProps<{ subPath?: string }>();

const router = createRouter({
  history: createMemoryHistory(props.subPath ?? '/'),
  routes: [ /* ... */ ],
});

// Sync on attribute changes (back/forward)
watch(() => props.subPath, (newPath) => {
  if (newPath) syncRouterFromHost(router, newPath);
});

// Sync on routeChanged events
const offRouteChanged = window.litiumExtension?.on<{ path: string }>(
  'routeChanged',
  (data) => { syncRouterFromHost(router, data.path); },
);
onUnmounted(() => offRouteChanged?.());
</script>
```

Navigate from page components:
```typescript
import { createNavigate } from '@litium/platform-extension-sdk/vue';
import { useRouter } from 'vue-router';

const router = useRouter();
const navigate = createNavigate(router);
navigate('/my-extension/orders/123');
```

Do NOT call `router.push()` alone — it updates Vue Router's memory state but not the host browser URL.

### Angular — MemoryLocationStrategy

Replace `PathLocationStrategy` with the generated `MemoryLocationStrategy`:

```typescript
providers: [
  { provide: LocationStrategy, useClass: MemoryLocationStrategy },
],
```

`MemoryLocationStrategy` intercepts `Router.navigate()` calls, translates them to `window.litiumExtension.navigate()`, and listens for `routeChanged` to call `Router.navigateByUrl()` back. Standard Angular navigation APIs work transparently.

```typescript
// Standard Angular navigation — MemoryLocationStrategy handles the bridge
this.router.navigate(['/orders', id]);
```

### Vanilla — Manual Routing

```typescript
connectedCallback() {
  this.#currentPath = this.getAttribute('sub-path') ?? '/';
  this.#render();
  this.#unsubscribe = window.litiumExtension?.on('routeChanged', ({ path }) => {
    this.#currentPath = path;
    this.#render();
  });
}
```

Navigate programmatically:
```typescript
window.litiumExtension.navigate('/my-extension/detail/123');
```

---

## Key Rules

1. **Never call `history.pushState` directly.** The host owns browser history.
2. **Never call `navigate()` inside a `routeChanged` handler.** This creates an infinite loop.
3. **Always read `sub-path` on first render** (`connectedCallback`). Without this, deep links and page refresh will always render the root page.
4. **Always unsubscribe** from `routeChanged` in your cleanup lifecycle hook.

## Multiple Settings Pages

Multiple `settings.menu.item` targets can point to different sub-paths of the same extension:

```json
{
  "target": "settings.menu.item",
  "name": "General Settings",
  "url": "/Litium/UI/settings/extensions/my-extension/general"
},
{
  "target": "settings.menu.item",
  "name": "Integration Config",
  "url": "/Litium/UI/settings/extensions/my-extension/integration-config"
}
```

Each menu item sets a different `sub-path`, which your router handles as separate routes.
