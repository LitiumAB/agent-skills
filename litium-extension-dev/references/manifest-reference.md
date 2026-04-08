# Extension Manifest Reference

Complete field-by-field reference for `extension.manifest.json` — the deployment descriptor that tells Litium how to load and register your extension.

## Minimal Example

```json
{
  "id": "my-extension",
  "name": "My Extension",
  "version": "1.0.0",
  "type": "ui",
  "targets": [
    {
      "target": "common.web-component",
      "name": "my-extension",
      "bundleUrl": "https://cdn.example.com/my-extension/extension.js",
      "customElementTag": "litium-ext-my-extension"
    }
  ]
}
```

---

## Top-Level Fields

### `id`

| | |
|---|---|
| **Type** | `string` |
| **Required** | Yes |
| **Format** | Lowercase kebab-case |

System-unique identifier. Used as the primary key when registering, updating, or deleting.

> Changing `id` after deployment creates a new extension entry rather than updating the existing one.

### `name`

| | |
|---|---|
| **Type** | `string` |
| **Required** | Yes |

Display name shown in the Extensions list. Prefix with `t:` to reference a translation key from `texts`:
```json
"name": "t:extensions.my-extension.name"
```

### `version`

| | |
|---|---|
| **Type** | `string` |
| **Required** | No |
| **Format** | Semantic version (e.g. `"1.2.3"`) |

Displayed in the Extensions list. Useful for auditing.

### `type`

| | |
|---|---|
| **Type** | `"ui" \| "admission_review"` |
| **Required** | Yes |

- `"ui"` — Admin-panel extension (bundles, menu items, API proxies)
- `"admission_review"` — Server-side webhook for admission review flows

All new backoffice UI extensions use `"ui"`.

### `description`

| | |
|---|---|
| **Type** | `string` |
| **Required** | No |

Short description displayed in the backoffice. Supports `t:` prefix for translation.

### `targets`

| | |
|---|---|
| **Type** | `ExtensionManifestTarget[]` |
| **Required** | No |

Array of target registrations. Each entry wires up one extension point.

### `texts`

| | |
|---|---|
| **Type** | `Record<string, Record<string, string>>` |
| **Required** | No |

Translations keyed by locale:

```json
"texts": {
  "en-US": {
    "extensions.my-extension.name": "My Order Widget",
    "extensions.my-extension.description": "Order management panel"
  },
  "sv-SE": {
    "extensions.my-extension.name": "Min orderwidget",
    "extensions.my-extension.description": "Orderhanteringspanel"
  }
}
```

---

## Targets Reference

Each object in the `targets` array must include a `target` field that selects the registered target definition.

### `common.web-component`

Registers a Web Component (Custom Element) bundle.

| Field | Type | Required | Description |
|---|---|---|---|
| `target` | `"common.web-component"` | Yes | Target type |
| `name` | `string` | Yes | Unique name for this target entry |
| `bundleUrl` | `string` | Yes | Absolute URL to the IIFE JS bundle. Must be HTTPS in production; `http://localhost` accepted during local dev. |
| `customElementTag` | `string` | Yes | Custom element tag. Must start with `litium-ext-`. |

```json
{
  "target": "common.web-component",
  "name": "my-extension",
  "bundleUrl": "https://cdn.example.com/my-extension/extension.js",
  "customElementTag": "litium-ext-my-extension"
}
```

### `settings.menu.item`

Adds an item to the Settings sidebar navigation.

| Field | Type | Required | Description |
|---|---|---|---|
| `target` | `"settings.menu.item"` | Yes | Target type |
| `name` | `string` | Yes | Translation key for the menu label |
| `url` | `string` | Yes | Path to activate when clicked |
| `permission` | `string` | No | Permission key required to see this item |

```json
{
  "target": "settings.menu.item",
  "name": "t:extensions.my-extension.menu",
  "url": "/Litium/UI/settings/extensions/my-extension",
  "permission": "extensions:my-extension:read"
}
```

### `{area}.menu.item`

Registers a web component panel in a backoffice area navigation panel. Replace `{area}` with: `customers`, `products`, `sales`, `media`, or `websites`.

| Field | Type | Required | Description |
|---|---|---|---|
| `target` | `"customers.menu.item"` etc. | Yes | Area-specific target |
| `name` | `string` | Yes | Display name in the area panel |
| `component` | `string` | Yes | Custom element tag. Must start with `litium-ext-`. |

```json
{
  "target": "products.menu.item",
  "name": "Pricing Rules",
  "component": "litium-ext-my-extension-panel-pricing-rules"
}
```

Panels are single-view — the host does not pass a `sub-path` attribute.

### `common.api.proxy`

Registers a transparent API proxy that forwards authenticated requests to your extension backend.

| Field | Type | Required | Description |
|---|---|---|---|
| `target` | `"common.api.proxy"` | Yes | Target type |
| `name` | `string` | Yes | Unique name |
| `url` | `string` | Yes | Base URL of your extension backend |

### `common.angular.module` *(deprecated)*

Angular Module Federation bundle. Use `common.web-component` for new extensions. See [migration.md](migration.md) to migrate.

---

## Local Dev Manifest

During development, point `bundleUrl` at the Vite dev server:

```json
{
  "target": "common.web-component",
  "name": "my-extension",
  "bundleUrl": "http://localhost:3000/src/index.ts",
  "customElementTag": "litium-ext-my-extension"
}
```

Switch `bundleUrl` back to the production CDN URL before deploying.

---

## Full Example

```json
{
  "id": "order-dashboard",
  "name": "t:extensions.order-dashboard.name",
  "version": "1.0.0",
  "type": "ui",
  "targets": [
    {
      "target": "common.web-component",
      "name": "order-dashboard",
      "bundleUrl": "https://cdn.example.com/order-dashboard/extension.js",
      "customElementTag": "litium-ext-order-dashboard"
    },
    {
      "target": "settings.menu.item",
      "name": "t:extensions.order-dashboard.menu",
      "ref_id": "system.settings",
      "url": "/Litium/UI/settings/extensions/order-dashboard"
    },
    {
      "target": "sales.menu.item",
      "name": "Order Dashboard",
      "component": "litium-ext-order-dashboard-panel-dashboard"
    }
  ],
  "texts": {
    "en-US": {
      "extensions.order-dashboard.name": "Order Dashboard",
      "extensions.order-dashboard.menu": "Order Dashboard"
    },
    "sv-SE": {
      "extensions.order-dashboard.name": "Orderöversikt",
      "extensions.order-dashboard.menu": "Orderöversikt"
    }
  }
}
```
