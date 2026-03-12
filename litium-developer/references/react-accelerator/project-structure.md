# Project Structure Reference

## Directory Layout

```
Litium.Storefront/
├── app/                      # Next.js App Router
│   ├── (content)/           # Route groups for layouts
│   │   ├── (StickyHeader)/  # Pages with sticky header
│   │   ├── (noNavigation)/  # Pages without navigation
│   │   └── (plain)/         # Minimal layout pages
│   ├── actions/             # Server Actions
│   ├── api/                 # API routes
│   ├── layout.tsx           # Root layout
│   └── error.tsx            # Error boundary
├── components/              # React components
│   ├── blocks/              # CMS block components (named: {TemplateId}Block.tsx)
│   ├── cart/                # Cart-related components
│   ├── checkout/            # Checkout flow components
│   │   ├── payments/        # Payment provider components
│   │   └── shipments/       # Shipment provider components
│   ├── elements/            # Base UI elements
│   ├── fields/              # Form field components
│   ├── form/                # Form components
│   ├── layouts/             # Layout wrapper components
│   ├── navigation/          # Navigation components
│   ├── products/            # Product display components
│   ├── productSearch/       # Product search/filter
│   ├── quickSearch/         # Quick search overlay
│   ├── search/              # Search results components
│   └── users/               # User/auth components
├── contexts/                # React Context providers
├── hooks/                   # Custom React hooks
├── models/                  # TypeScript interfaces/types
├── operations/              # GraphQL operations
│   └── fragments/           # GraphQL fragments
│       ├── blocks/          # Block type fragments ({TemplateId}Block fragment)
│       └── products/        # Product fragments
├── services/                # Business logic services
│   └── blockService.server.ts  # Block component registry
├── styles/                  # Global CSS/styles
├── utils/                   # Utility functions
├── public/                  # Static assets
├── e2e/                     # Playwright E2E tests
└── __mock__/               # Test mock data
```

## File Naming Conventions

### Components
| Pattern | Example | Description |
|---------|---------|-------------|
| `PascalCase.tsx` | `CartContent.tsx` | React component |
| `PascalCase.test.tsx` | `CartContent.test.tsx` | Jest unit test |
| `PascalCase.css` | `ImageGallery.css` | Component-specific CSS |

### Services
| Pattern | Example | Description |
|---------|---------|-------------|
| `*Service.server.ts` | `cartService.server.ts` | Server-only service |
| `*Service.client.ts` | `cartService.client.ts` | Client-only service |
| `*Service.ts` | `urlService.ts` | Isomorphic service |
| `*Service.test.ts` | `discountService.test.ts` | Service test |

### GraphQL Operations
| Pattern | Example | Description |
|---------|---------|-------------|
| `operations/fragments/*.ts` | `cart.ts` | Fragment definitions |
| `operations/fragments/blocks/*.ts` | `banner.ts` | Block-specific fragments |
| `operations/fragments/products/*.ts` | `product.ts` | Product fragments |

### Models
| Pattern | Example | Description |
|---------|---------|-------------|
| `models/*.ts` | `cart.ts` | TypeScript interfaces |

### Template Routes and Components
| Template Type | Folder/File Pattern | Example |
|---------------|---------------------|---------|
| Page template | `app/(content)/{Layout}/{Name}Page/[[...slug]]/page.tsx` | `ArticlePage/[[...slug]]/page.tsx` |
| Product template | `app/(content)/{Layout}/{Name}Product/[[...slug]]/page.tsx` | `ProductWithVariantsProduct/[[...slug]]/page.tsx` |
| Category template | `app/(content)/{Layout}/{Name}ProductCategory/[[...slug]]/page.tsx` | `CategoryProductCategory/[[...slug]]/page.tsx` |
| Block component | `components/blocks/{Name}Block.tsx` | `BannerBlock.tsx` |
| Block fragment | `operations/fragments/blocks/{name}.ts` | `banner.ts` |

## Route Group Conventions

Route groups `(name)` share layouts without affecting URL:

```
app/(content)/
├── (StickyHeader)/                              # Layout with sticky header
│   ├── ArticlePage/[[...slug]]/page.tsx        # Page template routes
│   ├── LandingPage/[[...slug]]/page.tsx
│   ├── LoginPage/[[...slug]]/page.tsx
│   ├── ProductWithVariantsProduct/[[...slug]]/page.tsx   # Product templates
│   └── (MyAccount)/                             # Nested account layout
│       └── OrderHistoryPage/[[...slug]]/page.tsx
├── (noNavigation)/                              # No navigation layout
│   ├── CheckoutPage/[[...slug]]/page.tsx
│   └── GlobalBlockPreview/[[...slug]]/page.tsx
└── (plain)/                                     # Minimal layout
    ├── CategoryProductCategory/[[...slug]]/page.tsx    # Category templates
    ├── ProductListPage/[[...slug]]/page.tsx
    └── SearchResultPage/[[...slug]]/page.tsx
```

### Template Folder Naming

**Critical:** Folder names must exactly match GraphQL type names for routing to work.

| Template Type | Naming Pattern | Example |
|---------------|----------------|---------|
| Page templates | `{TemplateId}Page` | `ArticlePage`, `LandingPage` |
| Product templates | `{TemplateId}Product` | `ProductWithVariantsProduct` |
| Category templates | `{TemplateId}ProductCategory` | `CategoryProductCategory` |

All template routes use `[[...slug]]` optional catch-all pattern to capture URL paths.

## Import Path Conventions

Use absolute imports (configured in `tsconfig.json`):
```typescript
// Preferred
import { Cart } from 'models/cart';
import { queryServer } from 'services/dataService.server';
import CartContent from 'components/cart/CartContent';

// Avoid relative imports for cross-directory
import { Cart } from '../../../models/cart'; // Don't do this
```

## Server vs Client Boundaries

### Server-Only Files
```typescript
// services/cartService.server.ts
import 'server-only';  // Enforce server-only usage

export async function get(): Promise<Cart> {
  // Can access headers, cookies, server resources
}
```

### Client-Only Files
```tsx
// components/cart/CartContent.tsx
'use client';  // First line, marks client boundary

import { useContext } from 'react';
```

### Isomorphic Files
```typescript
// utils/constants.ts
// No directive - runs in both environments
export const DiscountType = { ... };
```

## Generated Files

| File | Purpose | Script |
|------|---------|--------|
| `possibleTypes.json` | GraphQL union/interface types | `yarn generate-possible-types` |
| Block components | CMS block component stubs | `yarn generate-block-components` |

## Configuration Files

| File | Purpose |
|------|---------|
| `.env.local` | Local environment variables |
| `next.config.js` | Next.js configuration |
| `tsconfig.json` | TypeScript configuration |
| `jest.config.js` | Jest test configuration |
| `playwright.config.ts` | E2E test configuration |
| `eslint.config.mjs` | ESLint configuration |
