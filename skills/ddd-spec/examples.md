# ddd-spec — Code Examples

Detailed code examples referenced by SKILL.md. These show complete patterns
for atomic function specs, factory specs, and advanced factory body structures.

---

## Atomic function spec — complete example

```ts
// check-active.spec.ts
import type { FunctionSpec } from '../../../shared/spec'
import type { Cart, ActiveCart, CartId, CreatedAt } from '../../types'

export type CheckActiveFailure = 'cart_empty' | 'cart_confirmed' | 'cart_cancelled'
export type CheckActiveSuccess = 'cart-activity-confirmed'

// ── Test data ──────────────────────────────────────────────────────────────
const cartId = 'cart-1' as CartId
const createdAt = new Date('2024-01-01') as CreatedAt

const activeCart: Cart = { status: 'active', id: cartId, createdAt, items: [...], total: ... }
const emptyCart: Cart = { status: 'empty', id: cartId, createdAt }
const confirmedCart: Cart = { status: 'confirmed', id: cartId, createdAt, ... }
const cancelledCart: Cart = { status: 'cancelled', id: cartId, createdAt, ... }

// ── Spec ───────────────────────────────────────────────────────────────────
export const checkActiveSpec: FunctionSpec<
  Cart, ActiveCart, CheckActiveFailure, CheckActiveSuccess
> = {
  constraints: {
    cart_empty: {
      predicate: ({ input }) => input.status !== 'empty',
      examples: [{ when: emptyCart }],
    },
    cart_confirmed: {
      predicate: ({ input }) => input.status !== 'confirmed',
      examples: [{ when: confirmedCart }],
    },
    cart_cancelled: {
      predicate: ({ input }) => input.status !== 'cancelled',
      examples: [{ when: cancelledCart }],
    },
  },
  successes: {
    'cart-activity-confirmed': {
      condition: () => true,
      assertions: {
        'status-is-active': ({ output }) => output.status === 'active',
        'cart-id-preserved': ({ input, output }) => output.id === input.id,
      },
      examples: [{ when: activeCart, then: activeCart as ActiveCart }],
    },
  },
}
```

---

## Core factory spec — complete example

```ts
import type { FactorySpec } from '../../../shared/spec'
import { checkActiveSpec } from '../../shared/steps/check-active.spec'
import { checkProductInCartSpec } from '../../shared/steps/check-product-in-cart.spec'

// ── Test data ──────────────────────────────────────────────────────────────
const emptyCart = { ... }
const activeCartWithoutSneakers = { ... }
const inputReducing = { cart: cartWith4Sneakers, productId: sneakersId, quantity: 2 as Quantity }
const expectedReduced = { ... }
const inputRemoving = { cart: cartWith1Book, productId: bookId, quantity: 1 as Quantity }
const expectedRemoved = { ... }

// ── Spec ───────────────────────────────────────────────────────────────────
export const subtractQuantityCoreSpec: FactorySpec<
  CoreInput, CoreOutput, CoreFailure, CoreSuccess
> = {
  steps: {
    checkActive:       checkActiveSpec,
    checkProductInCart: checkProductInCartSpec,
  },
  failures: {
    cart_empty:  [{ when: { cart: emptyCart, productId: sneakersId, quantity: 2 as Quantity } }],
    not_in_cart: [{ when: { cart: activeCartWithoutSneakers, productId: sneakersId, quantity: 2 as Quantity } }],
    // each failure input passes all steps BEFORE the one that fails
  },
  successes: {
    'quantity-reduced': {
      condition: ({ output }) => output.status === 'active' && output.items.length > 0,
      examples: [{ when: inputReducing, then: expectedReduced }],
    },
    'item-removed': {
      condition: ({ input, output }) =>
        output.status === 'active' &&
        !output.items.some(i => i.productId === input.productId),
      examples: [{ when: inputRemoving, then: expectedRemoved }],
    },
    'cart-emptied': {
      condition: ({ output }) => output.status === 'empty',
      examples: [{ when: inputEmptying, then: expectedEmpty }],
    },
  },
}
```

---

## Shell factory spec — complete example

```ts
import type { ShellFactorySpec } from '../../../shared/spec'
import { parseCartIdSpec } from '../../shared/parse/parse-cart-id.spec'
import { subtractQuantityCoreSpec } from './core/subtract-quantity.spec'

// ── Test data ──────────────────────────────────────────────────────────────
const validShellInput = { cartId: 'cart-1', productId: 'prod-1', quantity: 1 }
const activeCart = { ... }
const expectedOutput = { ... }

// ── Spec ───────────────────────────────────────────────────────────────────
export const subtractQuantityShellSpec: ShellFactorySpec<
  ShellInput, ShellOutput, ShellFailure, ShellSuccess, Deps
> = {
  steps: {
    parseCartId:    parseCartIdSpec,
    parseProductId: parseProductIdSpec,
    parseQuantity:  parseQuantitySpec,
    core:           subtractQuantityCoreSpec,   // recursive — core is a step
  },
  deps: {
    findCartById: { failures: ['find_failed'] },
    saveCart:      { failures: ['save_failed'] },
  },
  failures: {
    // Step failures — real steps, crafted inputs, baseDeps
    not_a_string: [{ when: { cartId: 42, productId: 'prod-1', quantity: 1 } }],
    cart_empty:   [{ when: { cartId: 'cart-1', productId: 'prod-1', quantity: 1 } }],
    // baseDeps.findCartById returns an empty cart for this input
    // ...
  },
  successes: {
    'quantity-reduced': {
      condition: ({ output }) => output.status === 'active' && output.items.length > 0,
      examples: [{ when: validShellInput, then: expectedOutput }],
    },
  },
  // Shell-specific: dep propagation + baseDeps
  baseDeps: {
    findCartById: async (_id) => ({ ok: true, value: activeCart }),
    saveCart:      async (cart) => ({ ok: true, value: cart }),
  },
  depPropagation: {
    findCartById: { when: validShellInput, failsWith: 'find_failed' },
    saveCart:      { when: validShellInput, failsWith: 'save_failed' },
  },
}
```

---

## Strategy pattern — complete factory example

When behavior varies by data, declare a `Record<Tag, Handler>` step in `Steps`.
The factory dispatches by property lookup. No selection step, no branching.

```ts
// Strategy step — a Record of handlers, each with its own spec
type Steps = {
  validatePayment: (raw: unknown) => Result<ValidatedPayment, ValidationFailure, 'payment-validated'>
  process: Record<PaymentType, (payment: ValidatedPayment) => Result<ProcessedPayment, ProcessFailure, ProcessSuccess>>
  evaluateSuccessType: (args: { input: Input; output: Output }) => PaymentSuccess[]
}

const processPaymentFactory =
  (steps: Steps) =>
  (deps: Deps) =>
  async (input: Input): Promise<Result<Output, F, S>> => {
    // 1. validate payment input
    const payment = steps.validatePayment(input.payment)
    if (!payment.ok) return payment

    // 2. process the payment (dispatched by payment type)
    const processed = steps.process[payment.value.type](payment.value)
    if (!processed.ok) return processed

    // 3. persist the result
    const saved = await deps.savePayment(processed.value)
    if (!saved.ok) return saved

    // 4. evaluate success type
    const successType = steps.evaluateSuccessType({ input, output: saved.value })
    return { ok: true, value: saved.value, successType }
  }

// Steps wiring — each handler is a standalone function
const steps: Steps = {
  validatePayment,
  process: {
    instant:  processInstant,
    deferred: processDeferred,
  },
  evaluateSuccessType,
}
```

Each handler (`processInstant`, `processDeferred`) is a standalone function with
its own spec and tests. The factory doesn't know which handler runs — the dispatch
is a property access on the input's discriminant.
