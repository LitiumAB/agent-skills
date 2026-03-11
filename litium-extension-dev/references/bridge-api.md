# Bridge API Reference

Complete reference for `window.litiumExtension` — the API the Litium admin host injects into every Custom Element extension.

## TypeScript Setup

```typescript
import type {
  LitiumExtensionAPI,
  ExtensionContext,
  NotificationOptions,
  ExtensionEventType,
} from '@litium/platform-extension-sdk';

declare global {
  interface Window { litiumExtension: LitiumExtensionAPI; }
}
```

---

## Methods

### `navigate(path)`

Navigate within the Litium admin. Pushes a new browser history entry.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `path` | `string` | Yes | Full path from `/`, e.g. `/my-extension/detail` |

```typescript
window.litiumExtension.navigate('/my-extension/detail');
```

**Common mistakes:**
- Omitting the extension root prefix — the host needs the full path, e.g. `/my-extension/detail`, not `detail`.
- Calling `navigate()` inside a `routeChanged` handler — creates an infinite loop.

---

### `replaceUrl(path)`

Update the browser URL without pushing a new history entry. Use inside `routeChanged` handlers to sync the URL without clearing the forward stack.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `path` | `string` | Yes | Target path |

```typescript
window.litiumExtension.replaceUrl('/my-extension/detail');
```

**Common mistakes:**
- Using `navigate()` instead of `replaceUrl()` inside a `routeChanged` handler — pushes duplicate entries, breaks back button.

---

### `showNotification(options)`

Display a notification banner in the Litium admin UI.

| Field | Type | Required | Description |
|---|---|---|---|
| `options.message` | `string` | Yes | Notification text |
| `options.type` | `'success' \| 'info' \| 'warning' \| 'error'` | No | Severity (default: `'info'`) |
| `options.timeout` | `number` | No | Auto-dismiss ms (default: `5000`) |

```typescript
window.litiumExtension.showNotification({
  message: 'Order saved successfully.',
  type: 'success',
});

window.litiumExtension.showNotification({
  message: 'Validation failed — check required fields.',
  type: 'error',
  timeout: 10000,
});
```

---

### `fetch(input, init?)`

Authenticated HTTP request to the Litium backend. Automatically attaches CSRF tokens and auth headers. Mirrors the native `fetch` API signature.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `input` | `RequestInfo \| URL` | Yes | Use relative paths like `/Litium/api/...` |
| `init` | `RequestInit` | No | Standard fetch options |

**Returns:** `Promise<Response>`

```typescript
// GET
const res = await window.litiumExtension.fetch('/Litium/api/my-endpoint');
const data = await res.json();

// POST with JSON body
const res = await window.litiumExtension.fetch('/Litium/api/my-endpoint', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ orderId: '123' }),
});
```

**Critical:** Always use `window.litiumExtension.fetch()` instead of native `fetch()` for Litium API calls. Native `fetch` will fail CSRF validation and return 403 Forbidden.

---

### `getContext()`

Retrieve the current extension context.

**Returns:** `ExtensionContext`

```typescript
const context = window.litiumExtension.getContext();
console.log(context.language);        // "en-US"
console.log(context.channelSystemId); // "6fa459ea-ee8a-3ca4..."
console.log(context.websiteSystemId); // "..."
console.log(context.baseUrl);         // "https://my-litium.example.com"
```

**Common mistake:** Calling `getContext()` once at module level and caching forever. The context changes when the user switches channel or language — subscribe to `contextChanged` to stay in sync.

---

### `on(event, handler)`

Subscribe to a host event.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `event` | `ExtensionEventType` | Yes | Event name |
| `handler` | `(data: T) => void` | Yes | Callback |

**Returns:** `() => void` — an unsubscribe function. **Always call it in your cleanup lifecycle hook.**

```typescript
const unsubscribe = window.litiumExtension.on<{ path: string }>(
  'routeChanged',
  ({ path }) => { console.log('New path:', path); },
);

// Call in disconnectedCallback / onUnmounted / ngOnDestroy:
unsubscribe();
```

---

### `off(event, handler)`

Unsubscribe a specific handler. Prefer using the return value of `on()` instead.

---

## Events

### `routeChanged`

Fired when the browser navigates within your extension's URL prefix (back/forward buttons, deep links).

**Handler data:** `{ path: string }` — new sub-path relative to your extension root.

```typescript
window.litiumExtension.on<{ path: string }>('routeChanged', ({ path }) => {
  myRouter.replace(path);
});
```

### `contextChanged`

Fired when the user changes the active channel or website.

**Handler data:** `ExtensionContext` — the updated context.

```typescript
window.litiumExtension.on<ExtensionContext>('contextChanged', (newContext) => {
  loadData(newContext.channelSystemId);
});
```

### `languageChanged`

Fired when the user changes the UI language.

**Handler data:** `{ language: string }` — e.g. `"sv-SE"`.

```typescript
window.litiumExtension.on<{ language: string }>('languageChanged', ({ language }) => {
  i18n.setLocale(language);
});
```

---

## Type Reference

```typescript
interface ExtensionContext {
  language: string;          // UI language code, e.g. "en-US"
  channelSystemId: string;   // Currently selected channel
  websiteSystemId: string;   // Currently selected website
  baseUrl: string;           // Litium instance base URL
  [key: string]: unknown;    // Additional host-provided data
}

type ExtensionEventType = 'routeChanged' | 'contextChanged' | 'languageChanged';

interface NotificationOptions {
  message: string;
  type?: 'success' | 'info' | 'warning' | 'error';
  timeout?: number; // milliseconds, default 5000
}
```
