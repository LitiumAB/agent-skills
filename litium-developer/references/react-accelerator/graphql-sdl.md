# GraphQL SDL Reference

## Fetching the Schema

### Step 1: Get the Backend URL

Read `RUNTIME_LITIUM_SERVER_URL` from `.env.local`:
```bash
grep RUNTIME_LITIUM_SERVER_URL .env.local
# Example: RUNTIME_LITIUM_SERVER_URL=https://demoadmin.litium.com/
```

### Step 2: Construct SDL URL

```
SDL URL = ${RUNTIME_LITIUM_SERVER_URL}storefront.graphql?sdl
GraphQL Endpoint = ${RUNTIME_LITIUM_SERVER_URL}storefront.graphql
```

**Example:**
- Backend: `https://demoadmin.litium.com/`
- SDL: `https://demoadmin.litium.com/storefront.graphql?sdl`
- Endpoint: `https://demoadmin.litium.com/storefront.graphql`

### Step 3: Fetch the SDL

```bash
# Using curl
curl -s "$(grep RUNTIME_LITIUM_SERVER_URL .env.local | cut -d'=' -f2)storefront.graphql?sdl" > schema.graphql

# Or directly
curl -s "https://demoadmin.litium.com/storefront.graphql?sdl"
```

### Step 4: Analyze Schema

After fetching, search for queries, mutations, and types:
```bash
# List all queries
grep -E "^\s+(cart|product|page|content|checkout)" schema.graphql

# List all mutations
grep -E "mutation|Mutation" schema.graphql

# Find a specific type
grep -A 20 "type Cart {" schema.graphql
```

## Common Storefront API Operations

### Queries (Read Data)

| Query | Purpose | Common Use |
|-------|---------|------------|
| `content(url:)` | Get page content by URL | Page rendering |
| `cart` | Get current cart | Cart display |
| `checkout` | Get checkout state | Checkout flow |
| `productSearch` | Search/filter products | Product listing |
| `product(id:)` | Get single product | Product detail page |
| `order(id:)` | Get order details | Order confirmation |
| `me` | Get current user | User profile |
| `organizations` | Get B2B organizations | Organization selector |

### Mutations (Write Data)

| Mutation | Purpose | Common Use |
|----------|---------|------------|
| `addVariantToCart` | Add product to cart | Buy button |
| `removeVariantFromCart` | Remove from cart | Cart item removal |
| `updateVariantInCart` | Change quantity | Quantity selector |
| `setCheckoutAddress` | Set delivery address | Address form |
| `selectDeliveryOption` | Choose shipping | Delivery options |
| `selectPaymentOption` | Choose payment | Payment options |
| `placeOrder` | Complete checkout | Order submission |
| `login` | Authenticate user | Login form |
| `logout` | End session | Logout button |
| `addDiscountCode` | Apply discount | Discount input |

## Fragment Usage

### Existing Fragments

Fragments defined in `operations/fragments/` are auto-registered:

```typescript
// Use directly in queries - no import needed in GQL string
const GET_CART = gql`
  query GetCart {
    cart {
      ...Cart  # Uses CART_FRAGMENT from operations/fragments/cart.ts
    }
  }
`;
```

### Creating New Fragments

1. Create file in `operations/fragments/`:
```typescript
// operations/fragments/myFeature.ts
import { gql } from '@apollo/client';

export const MY_FEATURE_FRAGMENT = gql`
  fragment MyFeature on MyType {
    id
    name
    ...Image  # Can reference other fragments
  }
`;
```

2. Register in `services/apollo-client.ts`:
```typescript
import { MY_FEATURE_FRAGMENT } from 'operations/fragments/myFeature';

// Add to createFragmentRegistry()
createFragmentRegistry(
  // ... existing fragments
  MY_FEATURE_FRAGMENT,
);
```

## Type Generation

For TypeScript types matching the schema:

```bash
# Generate possibleTypes.json (union/interface mappings)
yarn generate-possible-types
```

This creates `possibleTypes.json` used by Apollo Client for cache normalization.

## GraphQL Playground

Access the interactive playground:
```
${RUNTIME_LITIUM_SERVER_URL}storefront.graphql
# Example: https://demoadmin.litium.com/storefront.graphql
```

Features:
- Schema browser (right sidebar)
- Query autocompletion
- Documentation explorer
- SDL download button

## API Documentation

Full Storefront API documentation:
- [Overview](https://litium.mintlify.app/apis/storefront/overview)
- [Queries](https://litium.mintlify.app/apis/storefront/queries)
- [Mutations](https://litium.mintlify.app/apis/storefront/mutations)
- [Types](https://litium.mintlify.app/apis/storefront/types)
