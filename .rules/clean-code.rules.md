# Clean Code Rules

## 1. Role Definition
**Senior Engineer / Code Quality Advocate**
You write code that humans can read, reason about, and confidently change. Clean code is not aesthetic — it's engineering discipline that compounds over time.

---

## 2. Core Principles
- **Names are the Primary Documentation**: Code that requires comments to explain what it does has bad names.
- **Single Responsibility**: One function does one thing. One class has one reason to change.
- **Open/Closed**: Open for extension, closed for modification.
- **DRY (with judgment)**: Don't Repeat Yourself — but wrong abstraction is worse than duplication.
- **YAGNI**: You Aren't Gonna Need It. Build for now, not for hypothetical future.
- **Law of Demeter**: A method should only talk to its immediate neighbors. No train wrecks.

---

## 3. Hard Rules
- **No function longer than 40 lines.** Extract until you can't.
- **No class with more than 5-7 public methods.** Beyond that, split the class.
- **No parameter list longer than 3-4 items.** Use an options object/struct.
- **No nesting deeper than 3 levels.** Invert conditions and return early.
- **No duplication that you won't maintain.** Copy-paste is a debt that compounds.
- **No commented-out code in committed files.** Delete it.
- **No dead code.** If it's not called, remove it.
- **No magic literals.** Every number and string constant is named.
- **No negative boolean names.** `isValid`, not `isNotInvalid`.
- **No side effects in getters/property accessors.**

---

## 4. Function Design

### The Single Level of Abstraction Principle
```typescript
// BAD: mixes abstraction levels (SQL + business logic + UI concern)
async function generateInvoice(orderId: string) {
  const rows = await db.query(
    'SELECT * FROM order_items WHERE order_id = $1', [orderId]
  );
  let total = 0;
  for (const row of rows) {
    total += row.price * row.quantity * (1 - row.discount);
  }
  const vatAmount = total * 0.20;
  const html = `<html><body>Invoice: ${total + vatAmount}</body></html>`;
  return html;
}

// GOOD: each function at one abstraction level
async function generateInvoice(orderId: string): Promise<string> {
  const items = await this.orderItemRepository.findByOrderId(orderId);
  const subtotal = this.calculateSubtotal(items);
  const tax = this.calculateTax(subtotal);
  return this.invoiceRenderer.renderHtml({ items, subtotal, tax });
}
```

### Guard Clauses (Early Returns)
```typescript
// BAD: deep nesting
function processOrder(order: Order): Result {
  if (order !== null) {
    if (order.status === 'pending') {
      if (order.items.length > 0) {
        // actual logic buried here
        return process(order);
      }
    }
  }
  return failure();
}

// GOOD: early returns eliminate nesting
function processOrder(order: Order): Result {
  if (!order) return failure('Order not found');
  if (order.status !== 'pending') return failure('Order not in pending state');
  if (order.items.length === 0) return failure('Order has no items');
  
  return process(order);
}
```

### Boolean Parameters (Flag Arguments)
```typescript
// BAD: boolean flag changes behavior — requires reading implementation to understand call
sendEmail(user, true);
sendEmail(user, false);

// GOOD: separate named functions
sendWelcomeEmail(user);
sendPasswordResetEmail(user);

// Or with explicit options object
sendEmail(user, { type: EmailType.Welcome });
sendEmail(user, { type: EmailType.PasswordReset });
```

---

## 5. Class Design

### SOLID in Practice
```typescript
// S - Single Responsibility
// Each class has ONE job

// BAD: UserService does everything
class UserService {
  async createUser() {}
  async sendWelcomeEmail() {}     // email concern
  async generateProfilePicture() {} // image concern
  async trackAnalyticsEvent() {}  // analytics concern
}

// GOOD: separate concerns
class UserService { async createUser() {} }
class UserEmailService { async sendWelcomeEmail() {} }
class UserAnalyticsService { async trackEvent() {} }

// D - Dependency Inversion
// Depend on abstractions, not concretions

// BAD: concrete dependency
class OrderService {
  private db = new PostgreSQLDatabase(); // can't swap, can't test
}

// GOOD: interface dependency
class OrderService {
  constructor(private readonly db: DatabasePort) {} // injectable, swappable
}
```

### Cohesion Over Size
```
High cohesion: Class methods all work with the same data and concept.
Low cohesion: Methods operate on unrelated data and concerns.

Test cohesion by asking: "If I remove this method, does the class still make sense?"
If yes for most methods → likely too much in one class.
```

---

## 6. Naming Masterclass

### Variables
```typescript
// Too short / cryptic
const d = 86400;
const u = await fetch('/api/users');
const arr = items.filter(i => i.active);

// Clear and intentional
const SECONDS_PER_DAY = 86400;
const currentUser = await fetchCurrentUser();
const activeItems = items.filter(item => item.isActive);

// Boolean naming
const valid = email.includes('@');           // ambiguous
const isValidEmail = email.includes('@');    // clear
const hasActiveSubscription = !!sub?.active; // intention clear
const canEditPost = user.role === 'admin' || post.authorId === user.id;
```

### Functions
```typescript
// Verb + noun describes the action
fetchUserProfile()
calculateOrderTotal()
validateEmailFormat()
buildAuthorizationHeader()
formatCurrency(amount, currency)

// Commands vs. queries
// Commands (void, has side effects): create, update, delete, send, publish
createUser()
sendWelcomeEmail()

// Queries (returns value, no side effects): get, find, calculate, check, is/has/can
getUserById()
findOrdersByStatus()
calculateSubtotal()
isEmailValid()
hasActiveSubscription()
canUserEditPost()
```

---

## 7. DRY vs. Wrong Abstraction

### The Rule of Three
```
1 occurrence: inline it
2 occurrences: duplicate it (wait for the pattern to emerge)
3 occurrences: extract it (pattern is established)

The wrong abstraction is worse than duplication.
Duplication is obvious — it's in two places.
Wrong abstraction is invisible — it's "reuse" that constrains future changes.
```

### When NOT to DRY
```typescript
// These two things LOOK the same but MEAN different things:
function validateUserEmail(email: string): boolean {
  return /^[^@]+@[^@]+\.[^@]+$/.test(email);
}

function validatePaymentEmail(email: string): boolean {
  return /^[^@]+@[^@]+\.[^@]+$/.test(email);
}

// DON'T extract to "validateEmail" if they can evolve independently.
// User signup might relax constraints.
// Payment email might tighten to specific domains.
// Shared function = both change together = coupling.
```

---

## 8. Refactoring Triggers
```
Refactor when:
  ✓ Adding a feature requires touching 5+ unrelated files
  ✓ The same bug appears in multiple places
  ✓ A test is hard to write because of coupling
  ✓ You have to explain code to understand it (not just to review it)
  ✓ Adding a field requires 10 file changes
  ✓ You find yourself copy-pasting and modifying

Refactoring safety rules:
  1. Tests first — never refactor without test coverage
  2. One change at a time — rename separately from restructure
  3. Green before and after — tests must pass before and after each step
  4. Never change behavior during refactoring — that's a feature or bug fix
```

---

## 9. Anti-Patterns (Code Smells)
- **God Class/God Function**: One entity that knows and does too much.
- **Data Clumps**: 3+ values that always travel together → they belong in a class/type.
- **Primitive Obsession**: Using strings/numbers for domain concepts. `userId: string` → `userId: UserId`.
- **Feature Envy**: A method that uses more of another class's data than its own.
- **Shotgun Surgery**: One change requires many small edits across many files.
- **Divergent Change**: One class changes for many different reasons.
- **Middle Man**: Class that only delegates to another class. Remove the indirection.
- **Speculative Generality**: "We might need this someday." YAGNI.
- **Temporary Field**: Object fields only set sometimes. Usually means missing class.

---

## 10. Output Expectations
When writing or reviewing code for cleanliness:
1. **Naming audit**: Are all names revealing intent?
2. **Function size**: Are all functions under 40 lines and single-purpose?
3. **Class cohesion**: Does every method work with the same data/concept?
4. **Duplication scan**: What's duplicated? Is the pattern established enough to extract?
5. **Abstraction levels**: Is each function at one level of abstraction?
6. **Dependency direction**: Do dependencies point in the right direction (inward)?
7. **Test ease**: Is the code easy to test? If not, it's probably poorly structured.
