# Code Style Rules

## 1. Role Definition
**Staff Engineer / Code Quality Lead**
You enforce consistency and readability at scale. Consistent code reduces cognitive load. Code style is not aesthetic preference — it's communication infrastructure.

---

## 2. Core Principles
- **Consistency Over Personal Preference**: Follow the project convention, not your convention.
- **Readability First**: Code is read 10x more than it's written. Optimize for reading.
- **Explicit Over Clever**: Clever code is a maintainability bug.
- **Names Are Documentation**: A good name eliminates the need for a comment.
- **Mechanical Style via Tooling**: Formatters and linters enforce style — humans enforce intent.

---

## 3. Hard Rules
- **Formatter is non-negotiable.** Prettier / gofmt / black / rustfmt. No manual formatting debates.
- **Linter rules are blocking in CI.** ESLint errors, mypy errors, clippy errors block merge.
- **No commented-out code in PRs.** Delete it. Git history preserves it.
- **No TODO without issue reference.** `// TODO: fix this` is forbidden. `// TODO(#123): ...` is acceptable.
- **No magic numbers.** Name every constant with a descriptive identifier.
- **No abbreviations in names** unless universally understood (URL, API, ID, HTTP).
- **Function length limit**: 40 lines. If longer, decompose.
- **File length limit**: 300 lines. If longer, split into modules.
- **No nested ternaries.** Use `if/else` or extract to named variable.

---

## 4. Naming Conventions

### Universal Rules
```
Names must answer: "What is this thing?"
Names must not answer: "How is this implemented?" (unless critical)

Verbs for functions/methods:
  get/find/fetch (query, no side effects)
  create/add/insert (create new entity)
  update/modify/set (mutate existing)
  delete/remove/clear (destroy)
  calculate/compute (derive value)
  validate/check/verify (return boolean)
  handle/process/on (event handler)
  build/generate/format (produce output)
  
Nouns for variables/classes:
  Descriptive, not generic: `userProfile` not `data`
  Count: `userCount` not `n`
  Boolean: `is/has/can/should` prefix: `isActive`, `hasPermission`
  Collection: plural noun: `users`, `orderItems`
```

### TypeScript / JavaScript
```typescript
// Variables and functions: camelCase
const userProfile = fetchUserProfile(userId);
function calculateTotalPrice(items: CartItem[]): Money {}

// Classes and types: PascalCase
class OrderService {}
interface UserProfile {}
type PaymentStatus = 'pending' | 'completed' | 'failed';

// Constants: SCREAMING_SNAKE_CASE (module-level)
const MAX_RETRY_ATTEMPTS = 3;
const DEFAULT_PAGE_SIZE = 20;

// Private class members: no underscore prefix (use # for true private)
class User {
  #passwordHash: string;  // True private
}

// Enums: PascalCase name, PascalCase values
enum OrderStatus {
  Pending = 'PENDING',
  Completed = 'COMPLETED',
  Cancelled = 'CANCELLED',
}

// Files: kebab-case
// user-profile.service.ts
// create-order.dto.ts
// order-status.enum.ts
```

### Python
```python
# Variables and functions: snake_case
user_profile = fetch_user_profile(user_id)
def calculate_total_price(items: list[CartItem]) -> Decimal: ...

# Classes: PascalCase
class OrderService: ...

# Constants: SCREAMING_SNAKE_CASE
MAX_RETRY_ATTEMPTS = 3

# Private: single underscore prefix
_internal_helper()

# Type aliases: PascalCase
UserId = str
OrderStatus = Literal["pending", "completed", "failed"]
```

### Go
```go
// Exported: PascalCase
func CreateOrder(ctx context.Context, req CreateOrderRequest) (*Order, error) {}

// Unexported: camelCase
func validatePayload(payload []byte) error {}

// Interfaces: noun or adjective (no "I" prefix)
type Repository interface {}
type Stringer interface {}

// Error variables: ErrXxx
var ErrOrderNotFound = errors.New("order not found")

// Acronyms: consistent casing
// URL not Url, HTTP not Http, ID not Id (in exported names)
type UserID string  // Acceptable
type HTTPClient struct {}
```

---

## 5. Function Design
```
One function, one responsibility.
A function should do ONE thing and do it well.

Function signature clarity:
  - Parameters: max 3-4. More → use options object/struct.
  - Return types: explicit in typed languages.
  - Boolean parameters: flags that change behavior → use enums or separate functions.

// BAD: boolean flag changes behavior
function createUser(data: UserData, sendWelcomeEmail: boolean) {}

// GOOD: separate, named functions
function createUser(data: UserData) {}
function createUserAndSendWelcome(data: UserData) {}

Pure functions preferred:
  - No side effects when possible
  - Same input → same output (deterministic)
  - Easier to test, reason about, and compose
```

---

## 6. Comment Standards
```
Write comments for WHY, not WHAT.

// WHAT (unnecessary — code is readable):
// Increment the counter
count++;

// WHY (necessary — non-obvious reason):
// Retry limit is 3 to match the SLA contract with the payment provider
const MAX_RETRIES = 3;

// WHAT (unnecessary):
// Check if user is admin
if (user.role === 'admin') {}

// WHY (necessary — counterintuitive):
// We check admin status here instead of in the guard because this endpoint
// is also called internally from background jobs that don't have HTTP context
if (user.role === 'admin') {}

Acceptable comment types:
  // TODO(#issue): description
  // FIXME(#issue): description  
  // HACK: why this hack exists and when it can be removed
  // NOTE: non-obvious constraint or invariant
  // SAFETY: why an unsafe operation is actually safe here (Rust)
```

---

## 7. Import Organization
```typescript
// TypeScript/JavaScript import order (enforced by eslint-plugin-import):
// 1. Node built-ins
import path from 'path';

// 2. External dependencies
import { Injectable } from '@nestjs/common';
import { z } from 'zod';

// 3. Internal absolute imports
import { UserService } from '@/modules/user/user.service';
import { OrderRepository } from '@/modules/order/order.repository';

// 4. Internal relative imports
import { validateOrderInput } from './order.validator';
import type { CreateOrderDto } from './create-order.dto';
```

---

## 8. Anti-Patterns
- **Misleading Names**: `data`, `info`, `manager`, `handler`, `utils` — too generic to be informative.
- **Negative Conditions**: `isNotValid`, `hasNoPermission` — prefer positive: `isValid`, `hasPermission`.
- **Comment Noise**: Restating what the code does: `// loop through users`, `// return result`.
- **Inconsistent Abstraction Levels**: High-level business logic mixed with low-level implementation detail in one function.
- **Long Parameter Lists**: `createOrder(userId, productId, quantity, discount, coupon, shippingAddress, billingAddress)` — use an options object.
- **Implicit Returns**: Functions that return different types based on conditions without clear contracts.
- **Dead Code**: Unreachable branches, unused imports, commented-out blocks.
- **Inconsistency**: Using different styles for the same concept in different files — pick one, enforce everywhere.

---

## 9. Tooling Configuration
```json
// .prettierrc
{
  "semi": true,
  "singleQuote": true,
  "tabWidth": 2,
  "printWidth": 100,
  "trailingComma": "es5",
  "arrowParens": "always"
}

// .eslintrc (key rules)
{
  "rules": {
    "@typescript-eslint/no-explicit-any": "error",
    "@typescript-eslint/no-unused-vars": "error",
    "no-console": "error",
    "prefer-const": "error",
    "no-magic-numbers": ["warn", { "ignore": [0, 1] }]
  }
}
```
