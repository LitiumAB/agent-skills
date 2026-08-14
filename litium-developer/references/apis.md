# Litium API Reference Guide

Overview of all REST APIs available to Litium partner developers, with links to OpenAPI specs and documentation.

## API Overview

| API | Type | Purpose | Spec File |
|-----|------|---------|-----------|
| Admin Web API | REST | Backend operations (products, orders, customers, media, etc.) | `assets/litium-admin-openapi.json` |
| Storefront API | GraphQL | Storefront operations (cart, checkout, products, pages) | Live schema via `/storefront.graphql?sdl` |
| Connect ERP | REST | ERP integration (orders, customers, products sync) | `assets/litium-connect-erp-openapi.json` |
| Connect Payment | REST | Payment provider integration callbacks | `assets/litium-connect-payment-openapi.json` |
| Connect Shipment | REST | Shipment provider integration callbacks | `assets/litium-connect-shipment-openapi.json` |
| Extension Management | REST | Manage installed Litium Extensions | `assets/litium-extension-management-openapi.json` |

---

## Admin Web API (REST)

Used for administrative operations: managing products, orders, customers, media, websites, etc.

- **Base URL**: `/litium/api/admin/`
- **Authentication**: OAuth 2.0 (client credentials or user token)
- **OpenAPI spec**: `assets/litium-admin-openapi.json`
- **Swagger UI**: `https://<your-domain>/Litium/swagger`

### Key Resource Groups

| Resource | Path |
|---|---|
| Products | `/litium/api/admin/products/` |
| Orders / Sales | `/litium/api/admin/sales/` |
| Customers | `/litium/api/admin/customers/` |
| Media | `/litium/api/admin/media/` |
| Websites | `/litium/api/admin/websites/` |
| Settings | `/litium/api/admin/settings/` |

### Example: Search Sales Orders

```powershell
$token = "<oauth-bearer-token>"
$headers = @{ Authorization = "Bearer $token"; "Content-Type" = "application/json" }
$body = '{ "take": 10, "skip": 0 }'

Invoke-RestMethod -Method Post `
  -Uri "https://mysite.com/litium/api/admin/sales/salesOrders/search" `
  -Headers $headers `
  -Body $body
```

### Authentication

Use OAuth 2.0 with the `/litium/oauth/token` endpoint:

```http
POST /litium/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&client_id=<id>&client_secret=<secret>&scope=litium.api
```

> **Note:** The `scope=litium.api` parameter is typically required. Client credentials are configured in Litium Back Office under Settings → System → OAuth.

---

## Storefront API (GraphQL)

Used by the React Accelerator and custom storefronts for all customer-facing operations.

- **Endpoint**: `/storefront.graphql`
- **Schema (SDL)**: `/storefront.graphql?sdl`
- **Authentication**: Session cookie (cart) or OAuth for user-specific operations
- **Docs**: https://docs.litium.dev/apis/storefront/overview

Key capabilities: products, pages, cart, checkout, wishlists, search, user account.

### Common Query Examples

```graphql
# Get product by slug
query {
  product(slug: "my-product") {
    name
    price { unitPrice { inclVat } }
    variants { id name }
  }
}

# Get current cart
query {
  cart {
    items { articleNumber quantity }
    grandTotal { inclVat }
  }
}
```

For cart and wishlist usage see `references/cart-and-wishlist.md`.  
For React Accelerator integration see `references/react-accelerator/overview.md`.

### How to Read the Live Schema

```bash
# Download SDL (--insecure for local dev with self-signed certs only)
curl https://localhost:5001/storefront.graphql?sdl --insecure -o schema.graphql

# Or use the Storefront playground
open https://localhost:5001/storefront.graphql
```

---

## Connect ERP API

Used to build ERP integrations (e.g., Dynamics 365, SAP, custom ERPs) that sync orders, customers, products, and inventory.

- **OpenAPI spec**: `assets/litium-connect-erp-openapi.json`
- **Docs**: https://docs.litium.dev/apps/erp-connect/overview
- **SDK**: `Litium.Connect.Erp.Sdk` NuGet package

The SDK provides event-based hooks and services for order export, customer sync, product import, and inventory updates.

---

## Connect Payment API

Used by custom payment app implementations to receive payment callbacks and process payment events.

- **OpenAPI spec**: `assets/litium-connect-payment-openapi.json`
- **Docs**: https://docs.litium.dev/apps/development-guide/overview
- **SDK**: `Litium.Apps.Payment` NuGet package

Use this when building a custom payment provider. Use the SDK to implement `IPaymentProvider` and handle callbacks.

---

## Connect Shipment API

Used by custom shipment app implementations to process shipping events and callbacks.

- **OpenAPI spec**: `assets/litium-connect-shipment-openapi.json`
- **Docs**: https://docs.litium.dev/apps/development-guide/overview
- **SDK**: `Litium.Apps.Shipment` NuGet package

---

## Extension Management API

Used to manage installed Litium Extensions (install, uninstall, list).

- **OpenAPI spec**: `assets/litium-extension-management-openapi.json`
- **Docs**: https://docs.litium.dev/platform/guides/back-office-ui-extensions

---

## How to Use the OpenAPI Specs

The JSON specs in `assets/` are standard OpenAPI 3.0 documents. To explore them:

```bash
# View available operations
cat assets/litium-admin-openapi.json | python3 -m json.tool | grep '"operationId"' | head -30

# Or install Swagger UI / use any OpenAPI viewer
npx @redocly/cli preview-docs assets/litium-admin-openapi.json
```

To make API calls based on a spec:

1. Find the operation in the spec (search for `operationId` or `path`)
2. Check required parameters and request body schema
3. Authenticate using the appropriate method (OAuth token or session)
4. Construct the request

---

## Useful Links

- Admin Web API guide: https://docs.litium.dev/apis/admin/overview
- Storefront API guide: https://docs.litium.dev/apis/storefront/overview
- ERP Connect: https://docs.litium.dev/apps/erp-connect/overview
- App Development (Payment/Shipment): https://docs.litium.dev/apps/development-guide/overview
- Back Office extensions: https://docs.litium.dev/platform/guides/back-office-ui-extensions
