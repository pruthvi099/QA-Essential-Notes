# Dependency Injection in Test Frameworks

## What It Is

Dependency injection (DI) is a design pattern where a component receives its dependencies (a driver, a config, an API client) from the outside — passed in — rather than creating them internally itself. In test frameworks, this is what already happens naturally through Playwright Test's fixture system (see [Fixtures](../02-automation-python-playwright/fixtures.md)) — this note makes that connection explicit and extends it to swapping implementations: real drivers vs. mocks, different environments, or test doubles for unit-testing framework code itself.

## Why It Matters

- A test/page object that creates its own dependencies internally (`new ApiClient()` inside a page object's constructor) is hard to control from outside — you can't swap in a mock API client for testing the framework's own logic, or point it at a different environment without changing the class itself.
- DI is what makes a framework's components genuinely composable and testable — fixtures already provide this in Playwright Test, but understanding the underlying DI principle explains *why* fixtures work the way they do, not just how to use the syntax.
- This is a more advanced framework design topic that shows up in senior SDET interviews specifically — connecting a familiar tool (fixtures) to the broader software engineering principle behind it (DI) demonstrates depth beyond tool-specific syntax knowledge.

## How It Works

**Without DI** — a component creates its own dependencies internally, making it inflexible:
```typescript
class OrderApiClient {
  private baseUrl = 'https://api.example.com';   // hardcoded, can't be swapped
  // ...
}
```

**With DI** — the dependency is passed in from outside, making the component flexible and testable:
```typescript
class OrderApiClient {
  constructor(private baseUrl: string) {}   // INJECTED — caller controls this
  // ...
}
```

**How Playwright Test's fixtures already implement DI:** a fixture provides a dependency (`page`, or a custom fixture like `authenticatedPage`) to a test — the test doesn't construct these itself, they're *injected* by the fixture system. This is DI, just using Playwright Test's own terminology and mechanism rather than a separate DI framework/library.

**Where explicit DI goes beyond what fixtures alone provide:** swapping a real dependency for a test double when testing the *framework's own code* (not application code) — e.g., testing that a retry helper correctly retries N times, without needing a real flaky network call to verify it.

## Example

**A framework utility designed with DI, making it independently testable:**
```typescript
// core/notification-service.ts
interface EmailSender {
  send(to: string, subject: string, body: string): Promise<void>;
}

export class TestFailureNotifier {
  constructor(private emailSender: EmailSender) {}   // INJECTED dependency

  async notifyOnFailure(testName: string, error: string) {
    await this.emailSender.send(
      'qa-team@example.com',
      `Test failed: ${testName}`,
      `Error details: ${error}`
    );
  }
}
```

**Testing the framework's OWN logic using a fake/mock implementation, instead of a real email service:**
```typescript
// Testing TestFailureNotifier's logic in isolation — this is testing
// the FRAMEWORK itself, distinct from testing the application under test
class FakeEmailSender implements EmailSender {
  public sentEmails: { to: string; subject: string; body: string }[] = [];

  async send(to: string, subject: string, body: string) {
    this.sentEmails.push({ to, subject, body });   // no real email sent
  }
}

test('notifier sends an email with the correct failure details', async () => {
  const fakeSender = new FakeEmailSender();
  const notifier = new TestFailureNotifier(fakeSender);   // injecting the FAKE

  await notifier.notifyOnFailure('checkout.spec.ts', 'Timeout waiting for element');

  expect(fakeSender.sentEmails).toHaveLength(1);
  expect(fakeSender.sentEmails[0].subject).toBe('Test failed: checkout.spec.ts');
});
```

**Recognizing Playwright Test's fixtures as DI in familiar form** — connecting the concept back to something already used throughout this repo:
```typescript
// This IS dependency injection — `page` and `authenticatedPage` are
// dependencies INJECTED into the test by the fixture system, not
// created by the test itself
test('checkout completes', async ({ page, authenticatedPage }) => {
  // The test receives what it needs; it doesn't construct these itself
});
```

## Production Considerations

- Apply explicit DI (interfaces, injected constructors) primarily to framework infrastructure code that benefits from independent testing or swappable implementations (notification services, retry logic, custom reporters) — most page objects and everyday test code get sufficient DI benefit "for free" through Playwright Test's fixture system alone, without needing additional formal DI patterns layered on top.
- Recognizing fixtures as a form of DI helps explain *why* fixture composition (a fixture depending on other fixtures, see [Fixtures](../02-automation-python-playwright/fixtures.md)) works the way it does — it's the same underlying principle, just expressed through Playwright Test's specific mechanism.
- Don't introduce a separate, heavyweight DI framework/library for a test automation project unless there's a genuine, complex need — Playwright Test's built-in fixture system already provides DI for the vast majority of test framework scenarios.

## Common Pitfalls

- Not recognizing that fixtures already provide DI, and either reinventing a parallel dependency-passing mechanism unnecessarily, or failing to leverage fixture composition where it would naturally solve a dependency-passing problem.
- Hardcoding dependencies inside framework utility classes (like the `OrderApiClient` example without DI) making them impossible to test in isolation or swap for a different environment/implementation.
- Over-engineering test automation code with a full, heavyweight DI framework/library when Playwright Test's built-in fixtures already provide sufficient dependency injection for the framework's actual needs.
- Confusing "testing the framework's own code" (where DI/mocking genuinely helps, as in the notifier example) with "testing the application under test" (where using real dependencies, per [Manual Testing vs. Automation Testing](../00-start-here/manual-testing-vs-automation-testing.md)'s general testing philosophy, is usually preferred over mocking).

## Interview Notes

- Be ready to explain dependency injection conceptually, and then connect it directly to Playwright Test's fixture system — this shows you understand the underlying principle, not just fixture syntax in isolation.
- Understand the distinction between testing application code (prefer real dependencies/integration) and testing framework infrastructure code (DI/mocking genuinely helps for isolated unit-style testing of the framework itself).
- Be able to describe a piece of framework code (a retry helper, a custom reporter, a notification service) that would benefit from explicit DI for its own testability.

## References

- [Martin Fowler — Inversion of Control Containers and the Dependency Injection Pattern](https://martinfowler.com/articles/injection.html)
- [Playwright — Test Fixtures (Node.js)](https://playwright.dev/docs/test-fixtures)