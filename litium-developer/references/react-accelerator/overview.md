# Litium React Accelerator Development

Next.js e-commerce storefront using Apollo Client 4.x + Litium Storefront GraphQL API.

## Workflow

1. **Understand the GraphQL API** — Fetch SDL: `references/react-accelerator/graphql-sdl.md`
2. **Know the structure** — Directory conventions: `references/react-accelerator/project-structure.md`
3. **Follow patterns** — Code patterns: `references/react-accelerator/code-patterns.md`
4. **Template routing** — Creating templates: `references/react-accelerator/template-routing.md`

## Template Routing (Critical for New Templates)

Litium uses proxy-based routing (`proxy.ts`) that maps URLs to Next.js routes based on CMS template names.

### Naming Convention Summary

| Template Type | Backend ID | GraphQL Type | Frontend Folder/File |
|---------------|------------|--------------|----------------------|
| Page | `Article` | `ArticlePage` | `app/(content)/.../ArticlePage/[[...slug]]/page.tsx` |
| Product | `ProductWithVariants` | `ProductWithVariantsProduct` | `app/(content)/.../ProductWithVariantsProduct/[[...slug]]/page.tsx` |
| Category | `Category` | `CategoryProductCategory` | `app/(content)/.../CategoryProductCategory/[[...slug]]/page.tsx` |
| Block | `Banner` | `BannerBlock` | `components/blocks/BannerBlock.tsx` |

**Pattern Rules:**
- Page templates: `{Name}` → `{Name}Page`
- Product templates: `{Name}` → `{Name}Product`
- Category templates: `{Name}` → `{Name}ProductCategory`
- Block templates: `{Name}` → `{Name}Block`

### Creating New Templates Checklist

**For new page/product/category templates:**
1. Create folder with exact GraphQL type name under `app/(content)/<LayoutGroup>/`
2. Use `[[...slug]]` catch-all route pattern
3. Add `page.tsx` with GraphQL query for the specific type

**For new block templates:**
1. Create `{Name}Block.tsx` in `components/blocks/` (or run `yarn generate-block-components`)
2. Register component in `services/blockService.server.ts`
3. Add fragment in `operations/fragments/blocks/`
4. Register fragment in `operations/fragments/blocks/allBlockTypes.ts`

See `references/react-accelerator/template-routing.md` for complete guide.

## Environment Configuration

GraphQL endpoint derived from `.env.local`:
```bash
RUNTIME_LITIUM_SERVER_URL=https://your-domain.com/  # Backend URL
# GraphQL endpoint: ${RUNTIME_LITIUM_SERVER_URL}storefront.graphql
# SDL download: ${RUNTIME_LITIUM_SERVER_URL}storefront.graphql?sdl
```

## Core Architecture

### Server vs Client Pattern

| Pattern | Use Case | Import |
|---------|----------|--------|
| `*.server.ts` | RSC, Server Actions | `import 'server-only'` |
| `*.client.ts` | Client Components | `'use client'` |
| `*.ts` (no suffix) | Isomorphic utilities | Pure functions |

### Data Flow

```
Server Component → queryServer() → Apollo Client → Litium Storefront API
Client Component → queryClient()/mutateClient() → Apollo Client → Proxy → Litium API
```

## GraphQL Operations Quick Reference

### Fragments Location
All fragments in `operations/fragments/`:
- `cart.ts` — Cart fields
- `checkout.ts` — Checkout, addresses
- `products/product.ts` — Full product data
- `products/productCard.ts` — Product list card
- `blocks/` — CMS block types
- `search.ts` — Search results

### Fragment Registration
Fragments auto-registered in `services/apollo-client.ts` via `createFragmentRegistry()`.

### Server Query Pattern
```typescript
import { gql } from '@apollo/client';
import { queryServer } from 'services/dataService.server';

const GET_DATA = gql`
  query GetData($id: ID!) {
    item(id: $id) { ...ItemFragment }
  }
`;

export async function getData(id: string) {
  const data = await queryServer({ query: GET_DATA, variables: { id } });
  return data.item;
}
```

### Client Mutation Pattern
```typescript
import { gql } from '@apollo/client';
import { mutateClient } from 'services/dataService.client';

const UPDATE_ITEM = gql`
  mutation UpdateItem($input: UpdateItemInput!) {
    updateItem(input: $input) { item { ...ItemFragment } }
  }
`;

export async function updateItem(input: UpdateItemInput) {
  const data = await mutateClient({ mutation: UPDATE_ITEM, variables: { input } });
  return data.updateItem.item;
}
```

## Component Conventions

### File Naming
- Component: `PascalCase.tsx`
- Test: `PascalCase.test.tsx`
- Index exports: Avoid barrel files (performance)

### Client Component
```tsx
'use client';
import { useContext } from 'react';
import { CartContext } from 'contexts/cartContext';

export function CartButton() {
  const { cart } = useContext(CartContext);
  return <button>{cart.productCount}</button>;
}
```

### Server Component (default)
```tsx
import { get } from 'services/websiteService.server';

export default async function Page() {
  const website = await get();
  return <div>{website.name}</div>;
}
```

## Testing Patterns

### Jest Unit Test
```typescript
import { render, screen } from '@testing-library/react';
import Component from './Component';

describe('Component', () => {
  it('renders correctly', () => {
    render(<Component />);
    expect(screen.getByText('expected')).toBeInTheDocument();
  });
});
```

### Data Attributes for Testing
Always add `data-testid` for e2e selection:
```tsx
<div data-testid="cart-content__empty-cart">...</div>
```

## Documentation

- [React Accelerator Docs](https://docs.litium.dev/accelerators/react-accelerator/overview)
- [Storefront API Docs](https://docs.litium.dev/apis/storefront/overview)
- [Routing Fundamentals](https://docs.litium.dev/accelerators/react-accelerator/routing-fundamentals)
- [How to create pages](https://docs.litium.dev/platform/guides/how-to-create-pages-in-react-accelerator)
- [How to create blocks](https://docs.litium.dev/platform/guides/how-to-create-blocks-in-react-accelerator)
