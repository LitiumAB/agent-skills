# Code Patterns Reference

## Service Layer Patterns

### Server Service Pattern

```typescript
// services/featureService.server.ts
import 'server-only';
import { queryServer } from './dataService.server';
import { gql } from '@apollo/client';
import { Feature } from 'models/feature';

const GET_FEATURE = gql`
  query GetFeature($id: ID!) {
    feature(id: $id) {
      ...Feature
    }
  }
`;

export async function get(id: string): Promise<Feature> {
  const data = await queryServer({
    query: GET_FEATURE,
    variables: { id },
  });
  return data.feature;
}
```

### Client Service Pattern

```typescript
// services/featureService.client.ts
import { gql } from '@apollo/client';
import { mutateClient, queryClient } from './dataService.client';
import { Feature } from 'models/feature';

export async function update(input: UpdateInput): Promise<Feature> {
  const data = await mutateClient({
    mutation: UPDATE_FEATURE,
    variables: { input },
  });
  return data.updateFeature.feature;
}

const UPDATE_FEATURE = gql`
  mutation UpdateFeature($input: UpdateFeatureInput!) {
    updateFeature(input: $input) {
      feature {
        ...Feature
      }
    }
  }
`;
```

## Context Provider Pattern

```tsx
// contexts/featureContext.tsx
'use client';
import { createContext, useState, ReactNode } from 'react';
import { Feature } from 'models/feature';
import { get as getFeature } from 'services/featureService.client';

type FeatureContextType = {
  feature: Feature;
  setFeature: (feature: Feature) => void;
  refresh: () => Promise<Feature>;
};

export const FeatureContext = createContext<FeatureContextType>({
  feature: defaultFeature,
  setFeature: () => {},
  refresh: async () => defaultFeature,
});

export default function FeatureContextProvider({
  value,
  children,
}: {
  value: Feature;
  children: ReactNode;
}) {
  const [feature, setFeature] = useState<Feature>(value);

  const refresh = async (): Promise<Feature> => {
    const updated = await getFeature();
    setFeature(updated);
    return updated;
  };

  return (
    <FeatureContext.Provider value={{ feature, setFeature, refresh }}>
      {children}
    </FeatureContext.Provider>
  );
}
```

## Component Patterns

### Client Component with Context

```tsx
// components/feature/FeatureDisplay.tsx
'use client';

import { useContext } from 'react';
import { FeatureContext } from 'contexts/featureContext';
import { useTranslations } from 'hooks/useTranslations';

export function FeatureDisplay() {
  const { feature } = useContext(FeatureContext);
  const t = useTranslations();

  return (
    <div data-testid="feature-display">
      <h2>{t('feature.title')}</h2>
      <p>{feature.description}</p>
    </div>
  );
}
```

### Server Component with Data Fetching

```tsx
// app/(content)/feature/page.tsx
import { get } from 'services/featureService.server';
import FeatureContextProvider from 'contexts/featureContext';
import { FeatureDisplay } from 'components/feature/FeatureDisplay';

export default async function FeaturePage() {
  const feature = await get();

  return (
    <FeatureContextProvider value={feature}>
      <FeatureDisplay />
    </FeatureContextProvider>
  );
}
```

## E-Commerce Specific Patterns

### Cart Operations

```typescript
// Add to cart with context refresh
import { add } from 'services/cartService.client';
import { CartContext } from 'contexts/cartContext';

const { setCart } = useContext(CartContext);

async function handleAddToCart(articleNumber: string, quantity: number) {
  const updatedCart = await add(articleNumber, quantity);
  setCart(updatedCart);
}
```

### Checkout Flow

```typescript
// Step-by-step checkout
import { 
  setCheckoutAddress,
  selectDeliveryOption,
  selectPaymentOption,
  placeOrder
} from 'services/checkoutService.client';

// 1. Set address
await setCheckoutAddress(addressInput);

// 2. Select delivery
await selectDeliveryOption(deliveryOptionId);

// 3. Select payment
await selectPaymentOption(paymentOptionId);

// 4. Place order
const order = await placeOrder();
```

### Product Search

```typescript
// services/productSearchService.client.ts
import { gql } from '@apollo/client';
import { queryClient } from './dataService.client';

export async function search(options: SearchOptions) {
  const data = await queryClient({
    query: PRODUCT_SEARCH,
    variables: {
      query: options.query,
      filters: options.filters,
      sort: options.sort,
      take: options.pageSize,
      skip: options.page * options.pageSize,
    },
  });
  return data.productSearch;
}

const PRODUCT_SEARCH = gql`
  query ProductSearch(
    $query: String
    $filters: [FilterInput!]
    $sort: SortInput
    $take: Int
    $skip: Int
  ) {
    productSearch(
      query: $query
      filters: $filters
      sort: $sort
      take: $take
      skip: $skip
    ) {
      ...ProductSearchResult
    }
  }
`;
```

## Translation Pattern

```tsx
// Using translations hook
import { useTranslations } from 'hooks/useTranslations';

function Component() {
  const t = useTranslations();
  
  return (
    <div>
      <h1>{t('page.title')}</h1>
      <p>{t('page.description', { name: 'John' })}</p>
    </div>
  );
}
```

## Form Pattern with react-hook-form

```tsx
'use client';

import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const schema = z.object({
  email: z.string().email(),
  quantity: z.number().min(1).max(99),
});

type FormData = z.infer<typeof schema>;

function FeatureForm() {
  const { register, handleSubmit, formState: { errors } } = useForm<FormData>({
    resolver: zodResolver(schema),
  });

  const onSubmit = async (data: FormData) => {
    // Handle submission
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('email')} />
      {errors.email && <span>{errors.email.message}</span>}
      <button type="submit">Submit</button>
    </form>
  );
}
```

## Testing Patterns

### Component Test

```tsx
// components/feature/Feature.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import { FeatureDisplay } from './FeatureDisplay';
import { FeatureContext } from 'contexts/featureContext';

const mockFeature = {
  id: '1',
  description: 'Test feature',
};

describe('FeatureDisplay', () => {
  it('renders feature description', () => {
    render(
      <FeatureContext.Provider value={{ 
        feature: mockFeature, 
        setFeature: jest.fn(),
        refresh: jest.fn(),
      }}>
        <FeatureDisplay />
      </FeatureContext.Provider>
    );

    expect(screen.getByText('Test feature')).toBeInTheDocument();
  });
});
```

### Service Test

```typescript
// services/featureService.test.ts
import { get } from './featureService.client';
import { queryClient } from './dataService.client';

jest.mock('./dataService.client');

describe('featureService', () => {
  it('returns feature data', async () => {
    (queryClient as jest.Mock).mockResolvedValue({
      feature: { id: '1', name: 'Test' },
    });

    const result = await get('1');
    
    expect(result).toEqual({ id: '1', name: 'Test' });
  });
});
```

## Price Formatting Pattern

```tsx
// Using FormattedPrice component
import FormattedPrice from 'components/FormattedPrice';

<FormattedPrice
  price={product.price}
  currency={cart.currency}
/>

// Server-side formatting
import FormattedPrice from 'components/FormattedPrice.server';
```

## Image Handling Pattern

```tsx
// Using the Image fragment
const PRODUCT_WITH_IMAGES = gql`
  query Product($id: ID!) {
    product(id: $id) {
      images(max: { height: 400, width: 400 }) {
        ...Image
      }
    }
  }
`;

// Rendering
import Image from 'next/image';

{product.images.map((img) => (
  <Image
    key={img.url}
    src={img.url}
    alt={img.alt || product.name}
    width={400}
    height={400}
  />
))}
```

## Error Handling Pattern

```tsx
'use client';

import { useTransition } from 'react';

function ActionButton() {
  const [isPending, startTransition] = useTransition();
  const [error, setError] = useState<string | null>(null);

  const handleAction = () => {
    setError(null);
    startTransition(async () => {
      try {
        await performAction();
      } catch (e) {
        setError(e instanceof Error ? e.message : 'An error occurred');
      }
    });
  };

  return (
    <>
      <button onClick={handleAction} disabled={isPending}>
        {isPending ? 'Loading...' : 'Submit'}
      </button>
      {error && <p className="text-red-500">{error}</p>}
    </>
  );
}
```
