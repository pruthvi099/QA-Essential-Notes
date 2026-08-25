# Designing a Scalable Page Object Hierarchy

## What It Is

This note extends [Page Object Model](../02-automation-python-playwright/page-object-model.md)'s single-page pattern into a full hierarchy suited to a large application: a **BasePage** providing shared functionality, **component objects** representing reusable UI pieces shared across many pages, and a deliberate strategy for avoiding the "God object" anti-pattern — one massive page class trying to represent an entire large page.

## Why It Matters

- A single-page-per-class approach works fine for a small app but breaks down at scale — a real application's checkout page might share a navigation bar, a cart drawer, and a footer with dozens of other pages; duplicating locators/methods for each shared piece across every page object is a massive, avoidable maintenance burden.
- The BasePage + component object pattern is what most real, large-scale Playwright/Selenium frameworks converge on independently — it's a well-established, expected pattern, not a niche technique, and its absence in a framework is a common code review finding.
- This is a natural, frequent follow-up question after "explain Page Object Model" in interviews — being able to go beyond the basic single-page example shows genuine experience building frameworks for real, complex applications, not just toy demos.

## How It Works

**BasePage** — an abstract/parent class holding functionality every page object needs: common navigation helpers, generic wait utilities, shared error-banner handling. Every specific page object extends it.

**Component objects** — represent a reusable UI piece (navigation bar, cart drawer, a modal dialog, a data table) that appears identically across many different pages. A component object is instantiated *within* multiple page objects, rather than each page object redefining the same locators.

**Avoiding the "God object" anti-pattern** — a single page object accumulating locators/methods for every section of a large, complex page (e.g., one giant `DashboardPage` covering fifteen unrelated widgets) becomes unwieldy. Breaking a large page into per-section component objects (even within a single logical "page") keeps each class focused and manageable.

## Example

**BasePage with shared functionality:**
```typescript
// pages/BasePage.ts
import { Page } from '@playwright/test';

export abstract class BasePage {
  constructor(protected page: Page) {}

  async waitForPageLoad() {
    await this.page.waitForLoadState('networkidle');
  }

  async getErrorBannerText(): Promise<string | null> {
    const banner = this.page.getByTestId('error-banner');
    return (await banner.isVisible()) ? banner.innerText() : null;
  }
}
```

**A reusable component object, used across many different pages:**
```typescript
// pages/components/NavBar.ts
import { Page, Locator } from '@playwright/test';

export class NavBar {
  readonly cartIcon: Locator;
  readonly cartBadge: Locator;
  readonly searchInput: Locator;

  constructor(private page: Page) {
    this.cartIcon = page.getByTestId('nav-cart-icon');
    this.cartBadge = page.getByTestId('nav-cart-badge');
    this.searchInput = page.getByPlaceholder('Search products');
  }

  async search(query: string) {
    await this.searchInput.fill(query);
    await this.searchInput.press('Enter');
  }

  async getCartCount(): Promise<string> {
    return this.cartBadge.innerText();
  }
}
```

**A specific page composing BasePage + the shared component, instead of redefining nav bar locators itself:**
```typescript
// pages/ProductPage.ts
import { Page, Locator } from '@playwright/test';
import { BasePage } from './BasePage';
import { NavBar } from './components/NavBar';

export class ProductPage extends BasePage {
  readonly navBar: NavBar;
  readonly addToCartButton: Locator;

  constructor(page: Page) {
    super(page);
    this.navBar = new NavBar(page);   // COMPOSITION — reused across many pages
    this.addToCartButton = page.getByRole('button', { name: 'Add to Cart' });
  }

  async addToCart() {
    await this.addToCartButton.click();
  }
}
```

```typescript
// tests/product.spec.ts — nav bar behavior is tested through the SAME
// component object, whether reached via ProductPage, CheckoutPage, etc.
test('cart count updates after adding a product', async ({ page }) => {
  const productPage = new ProductPage(page);
  await productPage.addToCart();

  expect(await productPage.navBar.getCartCount()).toBe('1');
});
```

**Breaking a large "God object" page into focused component objects:**
```typescript
// Instead of one giant DashboardPage with 15 unrelated widget locators,
// break it into focused component objects composed together
export class DashboardPage extends BasePage {
  readonly revenueWidget: RevenueWidget;
  readonly recentOrdersWidget: RecentOrdersWidget;
  readonly inventoryAlertsWidget: InventoryAlertsWidget;

  constructor(page: Page) {
    super(page);
    this.revenueWidget = new RevenueWidget(page);
    this.recentOrdersWidget = new RecentOrdersWidget(page);
    this.inventoryAlertsWidget = new InventoryAlertsWidget(page);
  }
}
```

## Production Considerations

- Introduce component objects as soon as a UI piece (nav bar, footer, a modal) is genuinely shared across more than one or two pages — waiting too long means locator duplication has already spread across many page objects, making the eventual refactor larger than if it had been done from the start.
- Keep BasePage genuinely generic — functionality specific to only a subset of pages doesn't belong there; a BasePage that accumulates page-specific logic over time defeats its own purpose as shared, universal functionality.
- Component objects composed into multiple page objects (as `NavBar` is above) mean a UI change to that shared component only requires updating one class — this compounding maintenance benefit is the core payoff that justifies the upfront structuring effort.

## Common Pitfalls

- Duplicating locators for a shared UI element (nav bar, footer) across every page object instead of extracting a reusable component object — a single UI change then requires updating many files instead of one.
- Letting a single page object grow into a "God object" covering an entire complex page's worth of unrelated sections, making the class hard to navigate and risky to change without affecting unrelated functionality.
- Putting page-specific logic into BasePage "because it's convenient," gradually turning a supposed shared base class into an inconsistent grab-bag that isn't genuinely reusable across all its subclasses.
- Not composing component objects (creating them fresh inline in every page object that needs them) instead of proper composition, losing the single-source-of-truth benefit the pattern is meant to provide.

## Interview Notes

- Be ready to design a BasePage + component object hierarchy for a described application, showing composition (not duplication) of shared UI elements — a very common, natural follow-up to basic Page Object Model questions.
- Understand the "God object" anti-pattern specifically and be able to describe how you'd refactor a large, unwieldy page object into focused, composed pieces.
- Be able to explain the concrete maintenance payoff of this pattern — a single component object update propagating to every page using it — with a specific example.

## References

- [Playwright — Page Object Models (Node.js)](https://playwright.dev/docs/pom)
- [Martin Fowler — PageObject](https://martinfowler.com/bliki/PageObject.html)