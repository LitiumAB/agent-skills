# Testing Extensions

Unit testing strategy, mocking `window.litiumExtension`, and CI configuration.

## Test Setup by Framework

| Framework | Test runner | Component testing library |
|---|---|---|
| React | Vitest | @testing-library/react |
| Vue | Vitest | @vue/test-utils |
| Angular | Jest + jest-preset-angular | Angular TestBed |

---

## Mocking `window.litiumExtension`

**Always mock `window.litiumExtension` before any component renders.** Components call bridge API methods during render.

### React / Vue (Vitest)

```typescript
import { vi } from 'vitest';

beforeEach(() => {
  (globalThis as unknown as { litiumExtension: unknown }).litiumExtension = {
    navigate: vi.fn(),
    replaceUrl: vi.fn(),
    showNotification: vi.fn(),
    getContext: vi.fn(() => ({
      language: 'en-US',
      channelSystemId: 'test-channel',
      websiteSystemId: 'test-website',
      baseUrl: 'http://localhost:5001',
    })),
    on: vi.fn(() => () => {}),   // returns a no-op unsubscribe function
    off: vi.fn(),
  };
});
```

### Angular (Jest)

```typescript
beforeEach(() => {
  (window as unknown as { litiumExtension: unknown }).litiumExtension = {
    navigate: jest.fn(),
    replaceUrl: jest.fn(),
    showNotification: jest.fn(),
    getContext: jest.fn(() => ({
      language: 'en-US',
      channelSystemId: '',
      websiteSystemId: '',
      baseUrl: '',
    })),
    on: jest.fn(() => () => {}),
    off: jest.fn(),
  };
});
```

---

## Example Tests

### Verify navigation (React)

```typescript
import { render } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { App } from './App';

it('calls navigate when the button is clicked', async () => {
  const { getByRole } = render(<App subPath="/pagea" />);
  await userEvent.click(getByRole('button', { name: /go to page b/i }));
  expect(window.litiumExtension.navigate).toHaveBeenCalledWith(
    expect.stringContaining('/pageb'),
  );
});
```

### Verify route rendering (React)

```typescript
it('renders PageA for subPath="/pagea"', () => {
  const { getByText } = render(<App subPath="/pagea" />);
  expect(getByText('Page A')).toBeDefined();
});

it('renders PageB for subPath="/pageb"', () => {
  const { getByText } = render(<App subPath="/pageb" />);
  expect(getByText('Page B')).toBeDefined();
});
```

### Verify Custom Element registration

```typescript
describe('Custom Element', () => {
  it('registers litium-ext-my-extension', async () => {
    await import('./index.js');
    expect(customElements.get('litium-ext-my-extension')).toBeDefined();
  });
});
```

### Verify context rendering (Angular)

```typescript
import { TestBed } from '@angular/core/testing';
import { PageAComponent } from './page-a.component';

describe('PageAComponent', () => {
  beforeEach(() => {
    // mock window.litiumExtension (see above)
    TestBed.configureTestingModule({
      declarations: [PageAComponent],
    });
  });

  it('displays the language from context', () => {
    const fixture = TestBed.createComponent(PageAComponent);
    fixture.detectChanges();
    expect(fixture.nativeElement.textContent).toContain('en-US');
  });
});
```

---

## Running Tests

```bash
# Unit tests (single run)
npm test

# Watch mode
npm run test:watch

# End-to-end tests (requires running Litium instance + dev server)
npm run test:e2e
```

E2E tests use [Playwright](https://playwright.dev/). Prerequisites:
- `npm run dev` running in one terminal
- A Litium instance with the dev extension registered

---

## Manual Testing Checklist

Run through before each deployment:

- [ ] Extension loads without console errors
- [ ] Default route renders the expected page
- [ ] All navigation links work
- [ ] `window.litiumExtension.navigate()` updates the browser URL
- [ ] Browser **Back** button returns to the previous page
- [ ] Browser **Forward** button works after navigating back
- [ ] Page refresh with a deep-link URL renders the correct page (not the root)
- [ ] `showNotification()` displays banner with correct message and type
- [ ] Admin API calls using an OAuth bearer token succeed (check Network tab for expected requests)
- [ ] `getContext()` returns correct values after switching channel/language
- [ ] Unmounting extension (navigating away) logs no memory-leak warnings

---

## CI Configuration (GitHub Actions)

```yaml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Configure npm registry
        run: echo "@litiumab:registry=https://registry.npmjs.org/" >> .npmrc

      # If registry requires auth:
      # - name: Authenticate
      #   run: echo "//registry.npmjs.org/:_authToken=${{ secrets.LITIUM_NPM_TOKEN }}" >> .npmrc

      - run: npm ci
      - run: npm run build
      - run: npm test
```

The `.npmrc` in scaffolded projects already contains the registry scope. In CI where `.npmrc` is in source control, no extra config is needed (unless token auth is required).
