# ddd-implement — Code Examples

Complete implementation patterns referenced by SKILL.md.

---

## Core factory — complete example

```ts
export type CoreSteps = {
  checkActive:         (cart: Cart) => Result<ActiveCart>
  checkProductInCart:   (input: { cart: ActiveCart; productId: ProductId }) => Result<ActiveCart>
  subtractQuantity:    (input: CoreInput) => Result<ActiveCart>
  // ...
  evaluateSuccessType: (args: { input: CoreInput; output: CoreOutput }) => CoreSuccess[]
}

export const subtractQuantityCoreFactory =
  (steps: CoreSteps) =>
  (input: CoreInput): Result<CoreOutput, CoreFailure, CoreSuccess> => {
    // 1. ensure cart is in active state
    const active = steps.checkActive(input.cart)
    if (!active.ok) return active

    // 2. find the target product in the cart
    const found = steps.checkProductInCart({ cart: active.value, productId: input.productId })
    if (!found.ok) return found

    // 3. reduce item quantity
    const subtracted = steps.subtractQuantity({ cart: found.value, productId: input.productId, quantity: input.quantity })
    if (!subtracted.ok) return subtracted

    // ... remaining domain steps

    // N. evaluate success type
    const successType = steps.evaluateSuccessType({ input, output: result })
    return { ok: true, value: result, successType }
  }

// Steps wiring
export const coreSteps: CoreSteps = { checkActive, checkProductInCart, subtractQuantity, ..., evaluateSuccessType }
export const subtractQuantityCore = subtractQuantityCoreFactory(coreSteps)
```

---

## Shell factory — complete example

```ts
export type ShellSteps = {
  parseCartId:    (raw: unknown) => Result<CartId>
  parseProductId: (raw: unknown) => Result<ProductId>
  parseQuantity:  (raw: unknown) => Result<Quantity>
  core:           (input: CoreInput) => Result<CoreOutput>
}

export type Deps = {
  findCartById: (id: CartId)               => Promise<Result<ActiveCart>>
  saveCart:     (cart: ActiveCart | EmptyCart) => Promise<Result<ActiveCart | EmptyCart>>
}

export const subtractQuantityShellFactory =
  (steps: ShellSteps) =>
  (deps: Deps) =>
  async (input: ShellInput): Promise<Result<ShellOutput, ShellFailure, ShellSuccess>> => {
    // 1. validate cart id
    const cartId = steps.parseCartId(input.cartId)
    if (!cartId.ok) return cartId

    // 2. validate product id
    const productId = steps.parseProductId(input.productId)
    if (!productId.ok) return productId

    // 3. validate quantity
    const quantity = steps.parseQuantity(input.quantity)
    if (!quantity.ok) return quantity

    // 4. fetch cart from persistence
    const cart = await deps.findCartById(cartId.value)
    if (!cart.ok) return cart

    // 5. core domain logic
    const result = steps.core({ cart: cart.value, productId: productId.value, quantity: quantity.value })
    if (!result.ok) return result

    // 6. persist result
    const saved = await deps.saveCart(result.value)
    if (!saved.ok) return saved

    // 7. evaluate success type (from core result, which already classified)
    return { ok: true, value: saved.value, successType: result.successType }
  }

export const shellSteps: ShellSteps = { parseCartId, parseProductId, parseQuantity, core: subtractQuantityCore }
export const makeSubtractQuantity = subtractQuantityShellFactory(shellSteps)
```

---

## evaluateSuccessType — patterns

### Direct implementation

```ts
export const evaluateSuccessType = (args: {
  input: CoreInput
  output: CoreOutput
}): CoreSuccess[] => {
  const { input, output } = args
  if (output.status === 'empty') return ['cart-emptied']
  if (!output.items.some(i => i.productId === input.productId)) return ['item-removed']
  return ['quantity-reduced']
}
```

### Core factory ending

```ts
// N. evaluate success type
const successType = steps.evaluateSuccessType({ input, output: result })
return { ok: true, value: result, successType }
```

### Shell forwarding from core

```ts
// N. core domain logic
const coreResult = steps.core({ cart: cart.value, productId: productId.value, quantity: quantity.value })
if (!coreResult.ok) return coreResult

// N+1. persist result
const saved = await deps.saveCart(coreResult.value)
if (!saved.ok) return saved

// Forward successType from core — shell doesn't reclassify
return { ok: true, value: saved.value, successType: coreResult.successType }
```

### Reusing spec condition predicates

```ts
export const evaluateSuccessType = (args: {
  input: CoreInput
  output: CoreOutput
}): CoreSuccess[] =>
  Object.entries(spec.successes)
    .filter(([_, s]) => s.condition(args))
    .map(([name]) => name as CoreSuccess)
```

---

## Strategy pattern — complete example

```ts
// ── The handlers — each is a standalone function with its own spec ──────────
export const processInstant = (payment: ValidatedPayment): Result<
  ProcessedPayment, InstantPaymentFailure, 'instant-processed'
> => {
  // ... instant payment logic
  return { ok: true, value: processed, successType: ['instant-processed'] }
}

export const processDeferred = (payment: ValidatedPayment): Result<
  ProcessedPayment, DeferredPaymentFailure, 'deferred-scheduled'
> => {
  // ... deferred payment logic
  return { ok: true, value: processed, successType: ['deferred-scheduled'] }
}

// ── Steps — strategy is a Record<Tag, Handler> ────────────────────────────
type Steps = {
  validatePayment: (raw: unknown) => Result<ValidatedPayment, ValidationFailure, 'payment-validated'>
  process: Record<PaymentType, (payment: ValidatedPayment) => Result<ProcessedPayment, ProcessFailure, ProcessSuccess>>
  evaluateSuccessType: (args: { input: Input; output: Output }) => PaymentSuccess[]
}

// ── The factory — linear, no branching ─────────────────────────────────────
const processPaymentFactory =
  (steps: Steps) =>
  (deps: Deps) =>
  async (input: Input): Promise<Result<Output, F, S>> => {
    // 1. validate payment
    const payment = steps.validatePayment(input)
    if (!payment.ok) return payment

    // 2. process the payment (dispatched by payment type)
    const processed = steps.process[payment.value.type](payment.value)
    if (!processed.ok) return processed

    // 3. persist
    const saved = await deps.savePayment(processed.value)
    if (!saved.ok) return saved

    // 4. evaluate success type
    const successType = steps.evaluateSuccessType({ input, output: saved.value })
    return { ok: true, value: saved.value, successType }
  }

// ── Steps wiring ───────────────────────────────────────────────────────────
const steps: Steps = {
  validatePayment,
  process: {
    instant:  processInstant,
    deferred: processDeferred,
  },
  evaluateSuccessType,
}
```
