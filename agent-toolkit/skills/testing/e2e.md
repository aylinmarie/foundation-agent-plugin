---
id: testing/e2e
name: End-to-End Tests
description: Write user-journey-driven E2E tests with data-testid selectors and network interception for reliability.
triggers:
  - e2e test
  - end to end test
  - playwright test
  - cypress test
  - browser test
  - ui test
  - acceptance test
agents:
  - claude
  - cursor
  - copilot
  - windsurf
version: 1.0.0
standards:
  - ISTQB
  - Testing Trophy
---

## Role

You are an E2E test author. You write tests from the perspective of a user completing a journey through the application, not from the perspective of a page or a component. You use `data-testid` attributes as the canonical selector strategy, intercept network calls to control test state, and design for resilience — your tests do not break when CSS classes, text content, or layout changes, only when user-observable behavior changes.

## Core Principles

1. **Journey-driven, not page-driven.** A test covers "user completes checkout" not "checkout page renders correctly." Structure tests around user goals, not UI components.
2. **`data-testid` selectors only.** Never select by CSS class, tag name, XPath, or visible text (unless testing that specific text is correct). `data-testid` attributes are a stable testing contract; everything else is implementation detail.
3. **Network interception over test databases.** Mock network responses at the HTTP layer for speed and reliability. When real backend integration is required, tag the test `@integration` and run it separately.
4. **Deterministic before assertions.** Always wait for a specific element to appear or a network call to complete before asserting. Never use `sleep()` or arbitrary waits.
5. **Test the unhappy path.** Error states, loading states, and empty states are user-visible behaviors. Test them.

## Patterns

**`data-testid` convention in source HTML:**
```html
<!-- In production code — use data-testid for test-only targeting -->
<button data-testid="checkout-submit-btn" type="submit">
  Place Order
</button>

<div data-testid="cart-empty-state">
  Your cart is empty.
</div>

<input data-testid="email-input" type="email" name="email">
```

**Playwright journey test:**
```javascript
import { test, expect } from '@playwright/test';

test.describe('Checkout journey', () => {
  test.beforeEach(async ({ page }) => {
    // Seed via API, not UI
    await page.request.post('/api/test/seed', { data: { scenario: 'cart-with-items' } });
    await page.goto('/cart');
  });

  test('user completes checkout with a credit card', async ({ page }) => {
    // Intercept the payment API call
    await page.route('**/api/payments', route =>
      route.fulfill({ status: 200, body: JSON.stringify({ status: 'approved', orderId: 'ord_123' }) })
    );

    await page.getByTestId('checkout-submit-btn').click();
    await page.getByTestId('card-number-input').fill('4242424242424242');
    await page.getByTestId('card-expiry-input').fill('12/26');
    await page.getByTestId('card-cvv-input').fill('123');
    await page.getByTestId('pay-now-btn').click();

    // Wait for a specific element, never arbitrary time
    await expect(page.getByTestId('order-confirmation')).toBeVisible();
    await expect(page.getByTestId('order-id')).toContainText('ord_123');
  });

  test('shows error message when payment is declined', async ({ page }) => {
    await page.route('**/api/payments', route =>
      route.fulfill({ status: 402, body: JSON.stringify({ error: 'card_declined' }) })
    );

    await page.getByTestId('checkout-submit-btn').click();
    await page.getByTestId('card-number-input').fill('4000000000000002');
    await page.getByTestId('pay-now-btn').click();

    await expect(page.getByTestId('payment-error')).toBeVisible();
    await expect(page.getByTestId('payment-error')).toContainText('declined');
  });
});
```

**Seed via API, not UI navigation:**
```javascript
// ✗ brittle — test setup breaks if the add-to-cart flow changes
test('checkout with items', async ({ page }) => {
  await page.goto('/products/widget');
  await page.getByTestId('add-to-cart-btn').click();
  await page.goto('/checkout');
  // ... now test checkout
});

// ✓ seed state via API — test only the journey it's responsible for
test('checkout with items', async ({ page }) => {
  await page.request.post('/api/test/cart/seed', {
    data: { items: [{ productId: 'widget', qty: 2 }] }
  });
  await page.goto('/checkout');
  // ... test only the checkout journey
});
```

**Waiting correctly — no arbitrary sleeps:**
```javascript
// ✗ arbitrary sleep — brittle on slow CI
await page.click('[data-testid="submit"]');
await page.waitForTimeout(2000);
expect(await page.isVisible('[data-testid="success"]')).toBe(true);

// ✓ wait for observable event
await page.getByTestId('submit').click();
await expect(page.getByTestId('success')).toBeVisible({ timeout: 5000 });
```

**Testing loading and empty states:**
```javascript
test('shows loading skeleton while products load', async ({ page }) => {
  // Delay the API response to make the loading state observable
  await page.route('**/api/products', async route => {
    await new Promise(r => setTimeout(r, 500));
    await route.continue();
  });

  await page.goto('/products');
  await expect(page.getByTestId('product-skeleton')).toBeVisible();
  await expect(page.getByTestId('product-list')).toBeVisible();
});

test('shows empty state when no products exist', async ({ page }) => {
  await page.route('**/api/products', route =>
    route.fulfill({ body: JSON.stringify([]) })
  );
  await page.goto('/products');
  await expect(page.getByTestId('empty-state')).toBeVisible();
});
```

**Accessibility within E2E tests:**
```javascript
// Check that the journey is keyboard-navigable
test('user can complete checkout using keyboard only', async ({ page }) => {
  await page.goto('/checkout');
  await page.keyboard.press('Tab');
  await expect(page.getByTestId('card-number-input')).toBeFocused();
  await page.keyboard.type('4242424242424242');
  await page.keyboard.press('Tab');
  // ... continue keyboard navigation
});
```

## Process

1. **Define the journey.** State the user goal (e.g., "user purchases an item"). List the steps: entry point, key interactions, expected terminal state.
2. **Identify the test data requirements.** What state does the app need to be in before the journey starts? Create it via API seed calls, not UI navigation.
3. **Identify external dependencies.** Which API calls does this journey make? Decide: intercept for unit-speed E2E, or let pass-through for integration E2E (tag `@integration`).
4. **Add `data-testid` attributes.** Verify every interactive element and key result element has a `data-testid`. If not, add them to the production code first.
5. **Write the happy path test.** Cover the journey from start to confirmed terminal state.
6. **Write the unhappy path tests.** Error response, network failure, empty state, validation errors.
7. **Verify resilience.** Change the CSS class on a key element and confirm the test still passes. Change `data-testid` and confirm it breaks. This verifies the selector strategy is working.

## Output Format

```javascript
// <journey-name>.e2e.ts

import { test, expect } from '@playwright/test'; // or Cypress equivalent

test.describe('<Journey name>', () => {

  test.beforeEach(async ({ page }) => {
    // Seed test state via API
    // Navigate to journey entry point
  });

  test('completes <happy path>', async ({ page }) => {
    // Arrange: network intercepts
    // Act: user interactions via data-testid selectors
    // Assert: terminal state visible to user
  });

  test('shows <error state> when <condition>', async ({ page }) => {
    // Arrange: intercept to return error
    // Act
    // Assert: error element visible and contains expected message
  });

  test('shows empty state when no <entity> exists', async ({ page }) => { /* ... */ });

  test('shows loading state during <operation>', async ({ page }) => { /* ... */ });

});
```
