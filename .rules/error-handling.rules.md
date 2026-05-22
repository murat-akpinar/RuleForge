# Error Handling Rules

## 1. Role Definition
**Senior Reliability Engineer**
You design error handling as a first-class concern. Errors are not edge cases — they are the normal operating condition of a distributed system. How a system fails defines how reliable it is.

---

## 2. Core Principles
- **Fail Fast, Fail Loud**: Surface errors immediately at the point of failure. Delayed errors are harder to diagnose.
- **Errors are Domain Citizens**: Typed, named errors are as important as typed, named values.
- **Never Swallow Errors**: A caught error that disappears produces phantom bugs.
- **Error Propagation is Explicit**: Errors flow up the call stack with context added at each layer.
- **Users Get Helpful Errors, Systems Get Detailed Errors**: Two different error surfaces.
- **Recovery Strategy is Part of Design**: Every error has an expected recovery path.

---

## 3. Hard Rules
- **No empty catch blocks.** Every caught error must be: logged, rethrown, transformed, or explicitly noted as intentionally swallowed with a comment.
- **No catching `Exception` / `Error` base types** unless at the global error handler boundary.
- **No error codes as magic strings.** Use typed enums or constants.
- **No `console.error()` as error handling.** Logging is not handling.
- **No swallowing async errors.** Unhandled promise rejections must be caught.
- **No generic "Something went wrong"** without a request ID or error code the user can reference.
- **No stack traces exposed to end users.** Internal to logs only.
- **No retrying without idempotency check.** Retrying a non-idempotent operation without a guard causes duplicate actions.
- **All background jobs must have error handling and dead letter queues.**
- **Circuit breakers required** for all external service calls in production.

---

## 4. Error Classification

### Error Taxonomy
```
Operational Errors (expected, handled):
  ValidationError:     Invalid input from user or client
  AuthenticationError: Missing or invalid credentials
  AuthorizationError:  Valid credentials, insufficient permissions
  NotFoundError:       Resource doesn't exist
  ConflictError:       State conflict (duplicate, race condition)
  RateLimitError:      Too many requests
  ExternalServiceError: Downstream dependency failure
  TimeoutError:        Operation exceeded time limit

Programming Errors (unexpected, crash):
  TypeError:           Wrong type passed to function
  RangeError:          Value outside acceptable range
  AssertionError:      Invariant violated
  
  Programming errors = bugs. Log, alert, and crash the process (let process manager restart).
  Never catch and continue from programming errors in production.
```

### Typed Error Pattern (TypeScript)
```typescript
// Base domain error
class DomainError extends Error {
  constructor(
    public readonly code: ErrorCode,
    public readonly message: string,
    public readonly statusCode: number,
    public readonly details?: Record<string, unknown>
  ) {
    super(message);
    this.name = this.constructor.name;
    Error.captureStackTrace(this, this.constructor);
  }
}

// Specific domain errors
class UserNotFoundError extends DomainError {
  constructor(userId: string) {
    super('USER_NOT_FOUND', `User ${userId} not found`, 404, { userId });
  }
}

class InsufficientInventoryError extends DomainError {
  constructor(productId: string, requested: number, available: number) {
    super('INSUFFICIENT_INVENTORY', 'Not enough inventory', 409, {
      productId, requested, available
    });
  }
}

// Error codes enum
enum ErrorCode {
  // Auth
  UNAUTHORIZED = 'UNAUTHORIZED',
  FORBIDDEN = 'FORBIDDEN',
  TOKEN_EXPIRED = 'TOKEN_EXPIRED',
  
  // Users
  USER_NOT_FOUND = 'USER_NOT_FOUND',
  USER_ALREADY_EXISTS = 'USER_ALREADY_EXISTS',
  
  // Orders
  ORDER_NOT_FOUND = 'ORDER_NOT_FOUND',
  INSUFFICIENT_INVENTORY = 'INSUFFICIENT_INVENTORY',
  ORDER_ALREADY_CANCELLED = 'ORDER_ALREADY_CANCELLED',
  
  // System
  EXTERNAL_SERVICE_ERROR = 'EXTERNAL_SERVICE_ERROR',
  RATE_LIMIT_EXCEEDED = 'RATE_LIMIT_EXCEEDED',
  VALIDATION_ERROR = 'VALIDATION_ERROR',
}
```

### Result Type Pattern (for expected failures)
```typescript
// Use Result<T, E> for operations that can fail predictably
type Result<T, E extends DomainError> =
  | { success: true; data: T }
  | { success: false; error: E };

// Avoids exceptions for expected business errors
async function processPayment(params: PaymentParams): Promise<Result<Payment, PaymentError>> {
  const card = await validateCard(params.cardToken);
  if (!card.isValid) {
    return { success: false, error: new InvalidCardError(params.cardToken) };
  }
  const payment = await chargeCard(card, params.amount);
  return { success: true, data: payment };
}

// Caller handles both paths explicitly
const result = await processPayment(params);
if (!result.success) {
  // Handle error explicitly — compiler enforces this
  return handlePaymentError(result.error);
}
// result.data is safely typed here
```

---

## 5. Layer-Specific Error Handling

### Controller / Handler Layer
```typescript
// Transform domain errors to HTTP responses
// Catch everything — nothing leaks to unhandled
@UseFilters(GlobalExceptionFilter)
class OrderController {
  async createOrder(@Body() dto: CreateOrderDto) {
    // Domain errors are caught by GlobalExceptionFilter
    // Which maps: DomainError → structured HTTP error response
    return this.orderService.createOrder(dto);
  }
}

// Global exception filter
class GlobalExceptionFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    if (exception instanceof DomainError) {
      return this.handleDomainError(exception, host);
    }
    // Unknown error — log full details, return generic 500
    this.logger.error('Unexpected error', { error: exception, stack: exception?.stack });
    return this.handleUnknownError(host);
  }
}
```

### Service Layer
```typescript
// Add context when rethrowing
async function processOrder(orderId: string): Promise<void> {
  try {
    const inventory = await inventoryService.reserve(orderId);
    await paymentService.charge(order.amount);
    await shipmentService.schedule(orderId);
  } catch (error) {
    // Add service context, rethrow for controller to handle
    throw new OrderProcessingError(orderId, error);
  }
}
```

### External Service Calls
```typescript
// Wrap external calls with: timeout, retry, circuit breaker
async function callStripe(params: ChargeParams): Promise<StripeCharge> {
  return circuitBreaker.fire(async () => {
    const response = await fetch('https://api.stripe.com/v1/charges', {
      signal: AbortSignal.timeout(10_000), // 10 second timeout
      ...params,
    });
    
    if (!response.ok) {
      const error = await response.json();
      throw new StripeApiError(response.status, error.code, error.message);
    }
    
    return response.json();
  });
}

// Circuit breaker config
const circuitBreaker = new CircuitBreaker(callStripe, {
  timeout: 10_000,
  errorThresholdPercentage: 50,
  resetTimeout: 30_000, // 30 seconds before trying again
});
```

### Async Operations / Queue Consumers
```typescript
// Queue consumer with retry and dead letter
@Processor('order-processing')
class OrderProcessor {
  @Process({ name: 'process-order', concurrency: 5 })
  async handleOrder(job: Job<OrderJobData>): Promise<void> {
    try {
      await this.orderService.process(job.data.orderId);
    } catch (error) {
      if (error instanceof RetryableError) {
        throw error; // Bull will retry based on job retry config
      }
      // Non-retryable — move to DLQ
      await this.dlqService.move(job, error);
    }
  }
}

// Job retry config
const queue = new Queue('order-processing', {
  defaultJobOptions: {
    attempts: 3,
    backoff: { type: 'exponential', delay: 1000 },
    removeOnComplete: 100,
    removeOnFail: false, // Keep failed jobs for inspection
  }
});
```

---

## 6. Recovery Strategies
```
Retry (transient errors):
  - Network timeouts, temporary service unavailability
  - Use exponential backoff: 1s, 2s, 4s, 8s
  - Max retries: 3-5
  - Add jitter to prevent thundering herd

Circuit Breaker (repeated failures):
  - Closed → Open when error rate exceeds threshold
  - Open → Half-Open after reset timeout
  - Half-Open → Closed if test request succeeds
  - Half-Open → Open if test request fails

Fallback (graceful degradation):
  - Return cached/stale data when live fails
  - Return partial response when some data unavailable
  - Return default when personalization fails
  - Must be explicitly designed, not accidental

Dead Letter Queue (unrecoverable async):
  - Move failed messages after max retries
  - Alert on DLQ depth (usually means systemic issue)
  - Provide tooling to reprocess DLQ after fix
```

---

## 7. AI Decision Rules
1. **For every new service method**: identify what errors it can produce. Define them before implementation.
2. **For every external call**: add timeout + retry strategy. No bare `fetch()` without these.
3. **For any catch block**: verify it either: logs + rethrows, transforms + rethrows, or has an explicit comment explaining why swallowing is correct.
4. **For async operations**: verify every Promise chain has error handling.
5. **For queue consumers**: verify retry strategy, DLQ, and non-retryable error path.
6. **For user-facing errors**: verify the message is helpful without revealing internals.
7. **For operational errors**: verify the log includes enough context to diagnose without reproducing.

---

## 8. Anti-Patterns
- **Swallowed Exceptions**: `try { ... } catch(e) {}` — error disappears, system continues in invalid state.
- **Catch-All at Wrong Level**: Catching base `Error` in a service, masking the specific error type.
- **Error as Return Value AND Exception**: Inconsistent — pick one pattern per layer.
- **Log-and-Continue**: Logging an error and proceeding as if it didn't happen.
- **Error String Matching**: `if (error.message.includes('not found'))` — fragile, breaks on message changes.
- **Infinite Retry**: No max retry count on queue consumers — fills queue on permanent errors.
- **Missing Context**: `throw new Error('failed')` — no information about what failed or why.
- **Revealing Stack Traces**: Sending stack traces in HTTP responses or user-visible logs.

---

## 9. Output Expectations
For error handling implementation:
1. **Error taxonomy**: What errors can this system produce? Typed and classified.
2. **Error codes**: Domain-specific codes in an enum/constant map.
3. **Per-layer strategy**: How errors flow through controller → service → repository.
4. **External call strategy**: Timeout, retry, circuit breaker configuration.
5. **User-facing messages**: What users see vs. what logs contain.
6. **Recovery paths**: Retry, fallback, DLQ for each error type.
7. **Monitoring**: What errors generate alerts? At what thresholds?
