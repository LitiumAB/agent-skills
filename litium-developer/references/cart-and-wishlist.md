# Cart, Checkout, and Wishlist

Reference for working with cart, checkout, and wishlists in Litium using the Storefront API and the React Accelerator.

---

## Approach Comparison

| Feature | Storefront API (GraphQL) | React Accelerator |
|---|---|---|
| Cart creation | `createCart` mutation | Automatic via `middleware.ts` |
| Add to cart | `addVariantToCart` mutation | `<BuyButton>` component |
| Cart state | Stored via cookie (cart ID) | `CartContext` via `useContext(CartContext)` |
| Checkout | Sequence of mutations | `CheckoutWizard` component |
| Wishlist | GraphQL mutations | See React wishlist guide |

---

## Cart — Storefront API

Docs: https://litium.mintlify.app/platform/guides/how-to-work-with-cart-and-checkout-using-the-storefront-api

### Create a Cart

```graphql
mutation {
  createCart {
    cart {
      rows { articleNumber quantity }
      currency { code }
      country { name code }
      showPricesIncludingVat
    }
  }
}
```

The returned cart is identified by a cookie. Store this on the client.

### Add a Variant

```graphql
mutation {
  addVariantToCart(input: {
    articleNumber: "my-article-001"
    quantity: 2
    additionalInfo: { key: "key", value: "value" }
  }) {
    cart {
      rows { rowId articleNumber quantity additionalInfo { key value } }
    }
  }
}
```

### Update Variant Quantity

```graphql
mutation {
  updateVariantInCart(input: {
    id: "rowId-value"
    quantity: 1
  }) {
    cart {
      rows { rowId articleNumber quantity }
      grandTotal
    }
  }
}
```

### Remove Variant

```graphql
mutation {
  removeVariantFromCart(input: { id: "rowId-value" }) {
    cart {
      rows { rowId articleNumber quantity }
      grandTotal
    }
  }
}
```

### Discount Codes

```graphql
mutation {
  addDiscountCodesToCart(input: { codes: ["PROMO10"] }) {
    cart { discountCodes }
  }
}

mutation {
  removeDiscountCodesFromCart(input: { codes: ["PROMO10"] }) {
    cart { discountCodes }
  }
}
```

### Query Cart

```graphql
query {
  cart {
    rows { rowId articleNumber quantity description }
    grandTotal
    totalVat
    productCount
  }
}
```

---

## Checkout — Storefront API

### 1. Create a Checkout Session

```graphql
mutation {
  createCheckoutSession(input: {
    checkoutFlowInfo: {
      checkoutPageUrl: "https://mysite.com/checkout"
      receiptPageUrl: "https://mysite.com/receipt"
      cancelPageUrl: "https://mysite.com/cancel"
      termUrl: "https://mysite.com/terms"
      allowSeparateShippingAddress: true
      customerType: PERSON
      shippingTags: ["free-shipping"]
      disablePaymentShippingOptions: false
    }
    notifications: {
      orderConfirmedUrl: "https://mysite.com/order-confirmed-webhook"
    }
  }) {
    checkout {
      checkoutFlowInfo { checkoutPageUrl receiptPageUrl }
    }
  }
}
```

`orderConfirmedUrl` must be an absolute URL registered in Litium Back Office. The system sends a `SalesOrderConfirmed` webhook event on order confirmation.

### 2. Update Checkout Details

```graphql
mutation ($address: OrderAddressInput, $customer: OrderCustomerDetailsInput) {
  updateCheckoutDetails(input: {
    billingAddress: $address
    shippingAddress: $address
    customer: $customer
  }) {
    checkout {
      customerDetails { firstName lastName email }
      billingAddress { ...Address }
    }
  }
}

fragment Address on OrderAddress {
  firstName lastName email address1 city zipCode country
}
```

### 3. Choose Payment and Shipping Options

```graphql
# Query available options first
query {
  checkout {
    shippingOptions { ...CheckoutOption }
    paymentOptions  { ...CheckoutOption }
    paymentHtmlSnippet  # widget HTML for Klarna, Adyen, etc.
  }
}

# Select options
mutation {
  updateCheckoutOptions(input: {
    shippingOptionId: "directshipment:expressPackage"
    paymentOptionId: "directpayment:DirectPay"
  }) {
    checkout {
      shippingOptions { ...CheckoutOption }
      paymentOptions  { ...CheckoutOption }
      paymentHtmlSnippet
    }
  }
}

fragment CheckoutOption on CheckoutOption {
  id name description price selected
}
```

`paymentHtmlSnippet` returns HTML+JS widget for providers like Klarna, Adyen, Svea, Qliro. Inject into DOM and execute scripts to render the widget.

### 4. Place Order

```graphql
mutation {
  placeOrder {
    receipt {
      order {
        rows { articleNumber quantity totalIncludingVat }
        customerDetails { firstName lastName }
        billingAddress { firstName lastName address1 city }
      }
      htmlSnippet  # Payment provider receipt snippet (if applicable)
    }
  }
}
```

### 5. Clear Cart After Order

```graphql
mutation {
  clearCart {
    cart { rows { rowId } grandTotal }
  }
}
```

---

## Cart — React Accelerator

Docs: https://litium.mintlify.app/accelerators/react/working-with-shopping-cart-and-checkout

### How It Works

- Cart is created automatically in `middleware.ts` if no cart cookie exists
- `CartContext` is loaded at the server in `app/layout.tsx` (Server Component)
- Client components access it via `useContext`:

```tsx
import { useContext } from 'react';
import { CartContext } from '@/contexts/CartContext';

function MyComponent() {
  const cartContext = useContext(CartContext);
  const { cart } = cartContext;
  return <div>Items: {cart?.productCount}</div>;
}
```

### Add to Cart

Use the built-in `BuyButton` component:

```tsx
import BuyButton from '@/components/BuyButton';

<BuyButton
  label="Add to cart"
  successLabel="Added to cart"
  articleNumber={articleNumber}
/>
```

### Checkout Flow

`CheckoutWizard` (`components/checkout/CheckoutWizard.tsx`) manages the 3-step checkout:

1. Enter delivery address
2. Choose delivery option
3. Payment

After each step, `checkoutService.client` sends a request to save data. Payment widgets for Iframe/Payment checkouts (Klarna, Adyen, etc.) are rendered via `PaymentWidget` using `paymentHtmlSnippet`.

### Payment Method Combinations

| Payment Type | Delivery Type | Supported Combination |
|---|---|---|
| Iframe checkout only (Klarna, Qliro, Svea) | Any | Only first iframe checkout rendered |
| Inline (DirectPay) | Inline (DirectShipment) | ✅ Combinable |
| Payment checkout (Adyen) | Inline | ✅ Combinable |
| Iframe checkout | Inline or Payment checkout | ❌ Iframe takes precedence |

---

## Wishlists — Storefront API

Docs: https://litium.mintlify.app/platform/guides/how-to-work-with-wishlists-using-the-storefront-api

> **Authentication required.** All wishlist mutations require the `litium_graphql_storefront` authorization policy. Anonymous users cannot create/manage wishlists.

### Create a Personal Wishlist

```graphql
mutation {
  createPersonalWishlist(input: {
    name: "My Favorites"
    category: "Holiday"
    products: [{ articleNumber: "product-001", quantity: 1 }]
  }) {
    wishlist { id name products { articleNumber quantity } }
    errors { ... on ValidationError { message } ... on Forbidden { message } }
  }
}
```

### Create an Organizational Wishlist (Shared)

```graphql
mutation {
  createOrganizationalWishlist(input: {
    name: "Office Supplies"
    isShared: true
    products: [{ articleNumber: "chair-001", quantity: 5 }]
  }) {
    wishlist { id name }
    errors { ... on Forbidden { message } }
  }
}
```

### Update / Add Products

```graphql
mutation {
  updateWishlist(input: {
    wishlistId: "wishlist-id-123"
    products: [
      { articleNumber: "new-product-456", quantity: 2 },
      { articleNumber: "remove-me-789", quantity: 0 }  # quantity 0 = remove
    ]
  }) {
    wishlist { id products { articleNumber quantity } }
  }
}
```

### Delete Wishlist

```graphql
mutation {
  deleteWishlist(input: { wishlistId: "wishlist-id-123" }) {
    errors { ... on ValidationError { message } }
  }
}
```

### Query All Wishlists

```graphql
query {
  wishlists(wishlistType: BOTH, sortBy: NAME, sortOrder: ASCENDING) {
    id name category createdAt modifiedAt
    products {
      articleNumber quantity
      productItem {
        ... on IProductItem {
          name
          images { url }
          price { includingVat currency { code } }
          stockStatus { inStock availableQuantity }
        }
      }
      additionalInfo { key value }
    }
  }
}
```

### Wishlist Type Enums

```graphql
enum WishlistTypeEnum { PERSONAL_ONLY  ORGANIZATION_ONLY  BOTH }
enum WishlistSortField { NAME  CREATED_AT  MODIFIED_AT }
enum WishlistSortOrder { ASCENDING  DESCENDING }
```

### React wishlist integration

See: https://litium.mintlify.app/accelerators/react/working-with-wishlists

---

## Useful Links

- Cart & Checkout (Storefront API): https://litium.mintlify.app/platform/guides/how-to-work-with-cart-and-checkout-using-the-storefront-api
- Cart & Checkout (React Accelerator): https://litium.mintlify.app/accelerators/react/working-with-shopping-cart-and-checkout
- Wishlists (Storefront API): https://litium.mintlify.app/platform/guides/how-to-work-with-wishlists-using-the-storefront-api
- Wishlists (React Accelerator): https://litium.mintlify.app/accelerators/react/working-with-wishlists
- Storefront API overview: https://litium.mintlify.app/apis/storefront/overview
