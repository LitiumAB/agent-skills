```markdown
# Template Routing Reference

## How Routing Works

Litium React Accelerator uses a proxy-based routing system (`proxy.ts`) that maps URLs to Next.js routes based on Litium templates.

### Routing Flow

1. Request arrives at `/some-page-url`
2. Proxy queries GraphQL `GET_CONTENT_TYPE` for the URL's `templateName`
3. URL is rewritten to `/${templateName}${originalPath}`
4. Next.js serves the matching route from `app/(content)/`

```
Request: /about → GraphQL returns templateName: "ArticlePage"
Rewrite: /ArticlePage/about → Next.js route: app/(content)/(StickyHeader)/ArticlePage/[[...slug]]/page.tsx
```

## Template Type Naming Conventions

Templates in Litium follow a consistent naming pattern that connects backend definitions to frontend routes.

### Page Templates

| Backend Template ID | GraphQL Type | Next.js Route Folder |
|---------------------|--------------|----------------------|
| `Article` | `ArticlePage` | `ArticlePage/` |
| `Landing` | `LandingPage` | `LandingPage/` |
| `Login` | `LoginPage` | `LoginPage/` |
| `Checkout` | `CheckoutPage` | `CheckoutPage/` |
| `SearchResult` | `SearchResultPage` | `SearchResultPage/` |
| `ProductList` | `ProductListPage` | `ProductListPage/` |
| `OrderConfirmation` | `OrderConfirmationPage` | `OrderConfirmationPage/` |

**Pattern**: Backend `{Name}` → GraphQL `{Name}Page` → Folder `{Name}Page/`

### Product Templates

| Backend Template ID | GraphQL Type | Next.js Route Folder |
|---------------------|--------------|----------------------|
| `ProductWithVariants` | `ProductWithVariantsProduct` | `ProductWithVariantsProduct/` |
| `ProductWithVariantsList` | `ProductWithVariantsListProduct` | `ProductWithVariantsListProduct/` |
| `ProductWithOneVariant` | `ProductWithOneVariantProduct` | `ProductWithOneVariantProduct/` |

**Pattern**: Backend `{Name}` → GraphQL `{Name}Product` → Folder `{Name}Product/`

### Category Templates

| Backend Template ID | GraphQL Type | Next.js Route Folder |
|---------------------|--------------|----------------------|
| `Category` | `CategoryProductCategory` | `CategoryProductCategory/` |

**Pattern**: Backend `{Name}` → GraphQL `{Name}ProductCategory` → Folder `{Name}ProductCategory/`

### Block Templates

| Backend Template ID | GraphQL Type | Component File |
|---------------------|--------------|----------------|
| `Banner` | `BannerBlock` | `BannerBlock.tsx` |
| `Product` | `ProductBlock` | `ProductsBlock.tsx` |
| `ProductsAndBanner` | `ProductsAndBannerBlock` | `ProductsAndBannerBlock.tsx` |
| `Slider` | `SliderBlock` | `SliderBlock.tsx` |
| `Video` | `VideoBlock` | `VideoBlock.tsx` |

**Pattern**: Backend `{Name}` → GraphQL `{Name}Block` → Component `{Name}Block.tsx`

## Creating New Templates

### Creating a New Page Template

1. **Backend**: Create `PageFieldTemplate` in `Litium.Accelerator/Definitions/Pages/`:
   ```csharp
   new PageFieldTemplate(PageTemplateNameConstants.MyCustomPage)
   {
       FieldGroups = new[] { /* field groups */ },
       Containers = new List<BlockContainerDefinition> { /* block containers */ }
   }
   ```

2. **Frontend**: Create route folder with exact GraphQL type name:
   ```
   app/(content)/(StickyHeader)/MyCustomPagePage/[[...slug]]/page.tsx
   ```

3. **Page Component Structure**:
   ```tsx
   import { gql } from '@apollo/client';
   import { Metadata } from 'next';
   import { queryServer } from 'services/dataService.server';
   import { createMetadata } from 'services/metadataService.server';

   export default async function Page(props: { params: Promise<any> }) {
     const params = await props.params;
     const content = await getContent({ params });
     return <div>{/* render content */}</div>;
   }

   export async function generateMetadata(props: { params: Promise<any> }): Promise<Metadata> {
     const params = await props.params;
     const content = await getContent({ params });
     return createMetadata(content.metadata);
   }

   async function getContent({ params }: { params: any }) {
     return (await queryServer({
       query: GET_CONTENT,
       url: params.slug?.join('/'),
     })).content;
   }

   const GET_CONTENT = gql`
     query GetMyCustomPagePage {
       content {
         ...Metadata
         ... on MyCustomPagePage {
           fields {
             # your custom fields
           }
         }
       }
     }
   `;
   ```

### Creating a New Product Template

1. **Backend**: Create `ProductFieldTemplate` in `Litium.Accelerator/Definitions/Products/`:
   ```csharp
   new ProductFieldTemplate(ProductTemplateNameConstants.MyProduct)
   {
       ProductFieldGroups = new[] { /* product fields */ },
       VariantFieldGroups = new[] { /* variant fields */ }
   }
   ```

2. **Frontend**: Create route folder:
   ```
   app/(content)/(StickyHeader)/MyProductProduct/[[...slug]]/page.tsx
   ```

3. **Query the product-specific type**:
   ```graphql
   query GetMyProductProduct {
     content {
       ...Metadata
       ...Product
       ... on MyProductProduct {
         fields {
           # custom product fields
         }
       }
     }
   }
   ```

### Creating a New Category Template

1. **Backend**: Create `CategoryFieldTemplate`:
   ```csharp
   new CategoryFieldTemplate(ProductTemplateNameConstants.MyCategory)
   {
       CategoryFieldGroups = new[] { /* category fields */ }
   }
   ```

2. **Frontend**: Create route folder:
   ```
   app/(content)/(plain)/MyCategoryProductCategory/[[...slug]]/page.tsx
   ```

### Creating a New Block Template

1. **Backend**: Create `BlockFieldTemplate` in `Litium.Accelerator/Definitions/Blocks/`:
   ```csharp
   new BlockFieldTemplate(BlockTemplateNameConstants.MyBlock)
   {
       CategorySystemId = pageCategoryId,
       Icon = "fas fa-cube",
       FieldGroups = new[] { /* block fields */ }
   }
   ```

2. **Frontend**: Create component in `components/blocks/` (or run `yarn run generate-block-components`):
   ```tsx
   // components/blocks/MyBlockBlock.tsx
   interface MyBlockBlockProps {
     systemId: string;
     fields: {
       // your block fields from GraphQL
     };
     priority?: boolean;
   }

   export default function MyBlockBlock(props: MyBlockBlockProps) {
     return <div>{/* render block */}</div>;
   }
   ```

3. **Register component** in `services/blockService.server.ts`:
   ```typescript
   import MyBlockBlock from 'components/blocks/MyBlockBlock';

   const Components: { [typename: string]: ... } = {
     // existing blocks...
     MyBlockBlock,
   };
   ```

4. **Add fragment** in `operations/fragments/blocks/`:
   ```typescript
   // operations/fragments/blocks/myBlock.ts
   import { gql } from '@apollo/client';

   export const MY_BLOCK_FRAGMENT = gql`
     fragment MyBlockBlock on MyBlockBlock {
       systemId
       fields {
         # your block fields
       }
     }
   `;
   ```

5. **Register fragment** in `operations/fragments/blocks/allBlockTypes.ts`:
   ```typescript
   import { MY_BLOCK_FRAGMENT } from './myBlock';

   export const ALL_BLOCK_TYPES_FRAGMENT = gql`
     fragment AllBlockTypes on IBlock {
       ... on IBlockItem { systemId }
       ...BannerBlock
       ...MyBlockBlock  // Add your block here
     }
     ${BANNER_BLOCK_FRAGMENT}
     ${MY_BLOCK_FRAGMENT}  // Include fragment
   `;
   ```

## Layout Groups

Choose the appropriate layout group for your template:

| Layout Group | Use Case | Features |
|--------------|----------|----------|
| `(StickyHeader)` | Standard pages, products | Sticky header, navigation |
| `(noNavigation)` | Checkout, previews | No navigation header |
| `(plain)` | Category listings, search | Minimal layout |
| `(MyAccount)` | User account pages | Account navigation sidebar |

## Route Catch-All Pattern

All template routes use the optional catch-all pattern `[[...slug]]`:

```
/ArticlePage/[[...slug]]/page.tsx
```

This captures:
- `/ArticlePage` → `params.slug = undefined`
- `/ArticlePage/about` → `params.slug = ['about']`
- `/ArticlePage/about/contact` → `params.slug = ['about', 'contact']`

## Error Handling Routes

Special routes for error pages:

| Route | Purpose |
|-------|---------|
| `ErrorPage/[[...slug]]/page.tsx` | 404 and 403 error display |
| `Redirect/[[...slug]]/route.ts` | Handle redirects from CMS |

## GraphQL ITemplateInfo Interface

The proxy relies on `ITemplateInfo` interface to get the template name:

```graphql
content {
  __typename           # Fallback if templateName is null
  ... on ITemplateInfo {
    templateName       # Primary source for routing
  }
}
```

If `templateName` is null, `__typename` is used directly for routing.

## Official Documentation

For step-by-step tutorials with screenshots:
- [Routing Fundamentals](https://docs.litium.dev/accelerators/react-accelerator/routing-fundamentals) — How URL-to-template mapping works
- [How to create pages](https://docs.litium.dev/platform/guides/how-to-create-pages-in-react-accelerator) — Creating page templates tutorial
- [How to create blocks](https://docs.litium.dev/platform/guides/how-to-create-blocks-in-react-accelerator) — Creating block templates tutorial
```
