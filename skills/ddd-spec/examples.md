# ddd-spec — Code Examples

Complete code examples referenced by SKILL.md. These show the v4 spec patterns
for all function types using `SpecFn` and `Spec<Fn>`.

---

## Parse function spec — complete example

Parse functions take `unknown` input. Errors accumulate. Output is the validated
domain type.

```ts
// parse-cart-id.spec.ts
import type { SpecFn, Spec } from '../../shared/spec-framework'

// -- Function contract --------------------------------------------------------

export type ParseCartIdFn = SpecFn<
    unknown,
    string,
    'not_a_string' | 'empty' | 'not_a_uuid',
    'cart-id-parsed'
>

// -- Spec ---------------------------------------------------------------------

export const parseCartIdSpec: Spec<ParseCartIdFn> = {
    shouldFailWith: {
        'not_a_string': {
            description: 'Should fail when input is not a string',
            examples: [
                { description: 'number is rejected',    whenInput: 42 },
                { description: 'null is rejected',      whenInput: null },
                { description: 'undefined is rejected', whenInput: undefined },
                { description: 'object is rejected',    whenInput: { id: 'x' } },
            ],
        },
        'empty': {
            description: 'Should fail when string is empty',
            examples: [
                { description: 'empty string is rejected', whenInput: '' },
            ],
        },
        'not_a_uuid': {
            description: 'Should fail when string is not a valid uuid',
            examples: [
                { description: 'plain text is rejected',   whenInput: 'not-a-uuid' },
                { description: 'partial uuid is rejected', whenInput: '550e8400-e29b-41d4' },
            ],
        },
    },

    shouldSucceedWith: {
        'cart-id-parsed': {
            description: 'Should parse and return a valid cart id',
            examples: [
                {
                    description: 'valid uuid v4 passes',
                    whenInput: '550e8400-e29b-41d4-a716-446655440000',
                    then: '550e8400-e29b-41d4-a716-446655440000',
                },
                {
                    description: 'uppercase uuid passes',
                    whenInput: '550E8400-E29B-41D4-A716-446655440000',
                    then: '550E8400-E29B-41D4-A716-446655440000',
                },
            ],
        },
    },

    shouldAssert: {
        'cart-id-parsed': {
            'output-equals-input': {
                description: 'Parsed value equals the input string',
                assert: (input, output) => input === output,
            },
        },
    },
}
```

---

## Step function spec — complete example

Step functions take a typed domain value. Errors accumulate. Asserts verify
properties of the narrowed output.

```ts
// check-active.spec.ts
import type { SpecFn, Spec } from '../../shared/spec-framework'
import type { Cart, ActiveCart } from '../types'

// -- Function contract --------------------------------------------------------

export type CheckActiveFn = SpecFn<
    Cart,
    ActiveCart,
    'cart_empty' | 'cart_confirmed' | 'cart_cancelled',
    'cart-activity-confirmed'
>

// -- Test data ----------------------------------------------------------------

const activeCart: Cart = {
    status: 'active', id: 'cart-4', customerId: 'c-1',
    items: [{ productId: 'p-1', qty: 3, unitPrice: 1299 }],
}
const activeCartMultiple: Cart = {
    status: 'active', id: 'cart-5', customerId: 'c-2',
    items: [
        { productId: 'p-1', qty: 1, unitPrice: 500 },
        { productId: 'p-2', qty: 5, unitPrice: 200 },
    ],
}
const emptyCart: Cart = { status: 'empty', id: 'cart-1' }
const confirmedCart: Cart = {
    status: 'confirmed', id: 'cart-2', customerId: 'c-1',
    items: [{ productId: 'p-1', qty: 2, unitPrice: 999 }],
}
const cancelledCart: Cart = {
    status: 'cancelled', id: 'cart-3', customerId: 'c-1',
    reason: 'user requested',
}

// -- Spec ---------------------------------------------------------------------

export const checkActiveSpec: Spec<CheckActiveFn> = {
    shouldFailWith: {
        'cart_empty': {
            description: 'Should fail when cart has no items',
            examples: [
                { description: 'empty cart is rejected', whenInput: emptyCart },
            ],
        },
        'cart_confirmed': {
            description: 'Should fail when cart is already confirmed',
            examples: [
                { description: 'confirmed cart is rejected', whenInput: confirmedCart },
            ],
        },
        'cart_cancelled': {
            description: 'Should fail when cart has been cancelled',
            examples: [
                { description: 'cancelled cart is rejected', whenInput: cancelledCart },
            ],
        },
    },

    shouldSucceedWith: {
        'cart-activity-confirmed': {
            description: 'Should confirm the cart is active and return it narrowed',
            examples: [
                {
                    description: 'active cart with one item passes through',
                    whenInput: activeCart,
                    then: activeCart as ActiveCart,
                },
                {
                    description: 'active cart with multiple items passes through',
                    whenInput: activeCartMultiple,
                    then: activeCartMultiple as ActiveCart,
                },
            ],
        },
    },

    shouldAssert: {
        'cart-activity-confirmed': {
            'status-is-active': {
                description: 'Output cart status is active',
                assert: (_input, output) => output.status === 'active',
            },
            'id-preserved': {
                description: 'Cart id is unchanged',
                assert: (input, output) => output.id === input.id,
            },
            'items-unchanged': {
                description: 'Cart items are passed through unmodified',
                assert: (input, output) => {
                    if (input.status !== 'active') return false
                    return JSON.stringify(output.items) === JSON.stringify(input.items)
                },
            },
        },
    },
}
```

---

## Composed factory spec — complete example (shell with steps)

Factory specs declare `steps` for algorithm visibility and failure inheritance.
Only overrides and own failures are declared — inherited failures resolve at runtime.

```ts
// remove-item.spec.ts
import type { SpecFn, Spec, StepInfo } from '../../shared/spec-framework'
import { parseCartIdSpec } from '../shared/parse/parse-cart-id.spec'
import { checkActiveSpec } from '../shared/steps/check-active.spec'
import type { ActiveCart } from '../types'

// -- Function contract --------------------------------------------------------

type RemoveItemInput = { cartId: unknown; productId: string }

export type RemoveItemFn = SpecFn<
    RemoveItemInput,
    ActiveCart,
    // Composed: parseCartId failures + checkActive failures + own failures
    'not_a_string' | 'empty' | 'not_a_uuid'
    | 'cart_empty' | 'cart_confirmed' | 'cart_cancelled'
    | 'cart_not_found' | 'product_not_in_cart',
    'item-removed'
>

// -- Algorithm ----------------------------------------------------------------

const steps: StepInfo[] = [
    { name: 'parseCartId', type: 'step', description: 'Validate and parse the raw cart id', spec: parseCartIdSpec },
    { name: 'findCart',    type: 'dep',  description: 'Fetch cart from persistence by id' },
    { name: 'checkActive', type: 'step', description: 'Verify cart is in active state',    spec: checkActiveSpec },
    { name: 'removeItem',  type: 'step', description: 'Remove the product from cart items' },
    { name: 'saveCart',    type: 'dep',  description: 'Persist the updated cart' },
]

// -- Test data ----------------------------------------------------------------

const CART_ID = '550e8400-e29b-41d4-a716-446655440000'

// -- Spec — behavioral contract + algorithm -----------------------------------

export const removeItemSpec: Spec<RemoveItemFn> = {
    steps,

    shouldFailWith: {
        // Override an inherited group — add an integration example
        'not_a_string': {
            description: 'Should fail when cart id is not a string',
            examples: [
                {
                    description: 'numeric cart id is rejected through the full pipeline',
                    whenInput: { cartId: 42, productId: 'p-1' },
                },
            ],
        },

        // Own failures — steps without specs, or deps
        'cart_not_found': {
            description: 'Should fail when cart does not exist in storage',
            examples: [
                {
                    description: 'non-existent cart id is rejected',
                    whenInput: { cartId: 'aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee', productId: 'p-1' },
                },
            ],
        },
        'product_not_in_cart': {
            description: 'Should fail when product is not in the cart',
            examples: [
                {
                    description: 'unknown product id is rejected',
                    whenInput: { cartId: CART_ID, productId: 'p-99' },
                },
            ],
        },

        // Inherited failures NOT declared here:
        //   'empty'          — covered by parseCartId
        //   'not_a_uuid'     — covered by parseCartId
        //   'cart_empty'     — covered by checkActive
        //   'cart_confirmed' — covered by checkActive
        //   'cart_cancelled' — covered by checkActive
        // These are auto-resolved from steps at runtime by testSpec()
    },

    shouldSucceedWith: {
        'item-removed': {
            description: 'Should remove the item from the cart',
            examples: [
                {
                    description: 'removes product from active cart',
                    whenInput: { cartId: CART_ID, productId: 'p-1' },
                    then: {
                        status: 'active', id: CART_ID, customerId: 'c-1',
                        items: [{ productId: 'p-2', qty: 3, unitPrice: 500 }],
                    },
                },
            ],
        },
    },

    shouldAssert: {
        'item-removed': {
            'product-no-longer-in-cart': {
                description: 'The removed product should not appear in the cart items',
                assert: (input, output) => !output.items.some(i => i.productId === input.productId),
            },
            'other-items-unchanged': {
                description: 'Other items in the cart should be unmodified',
                assert: (_input, output) => output.items.length > 0,
            },
        },
    },
}
```

---

## Core factory spec with multiple success types

Core factories often have multiple success types. Each success type has its own
condition, determined by `evaluateSuccessType` in the implementation.

```ts
// subtract-quantity/core/subtract-quantity.spec.ts
import type { SpecFn, Spec, StepInfo } from '../../../shared/spec-framework'
import { checkActiveSpec } from '../../shared/steps/check-active.spec'
import type { ActiveCart, EmptyCart, CartItem, ProductId, Quantity } from '../../types'

// -- Function contract --------------------------------------------------------

type CoreInput = { cart: ActiveCart; productId: ProductId; quantity: Quantity }
type CoreOutput = ActiveCart | EmptyCart

export type SubtractQuantityCoreFn = SpecFn<
    CoreInput,
    CoreOutput,
    'cart_empty' | 'cart_confirmed' | 'cart_cancelled'
    | 'product_not_in_cart' | 'insufficient_quantity',
    'quantity-reduced' | 'item-removed' | 'cart-emptied'
>

// -- Algorithm ----------------------------------------------------------------

const steps: StepInfo[] = [
    { name: 'checkActive',         type: 'step', description: 'Verify cart is active',             spec: checkActiveSpec },
    { name: 'checkProductInCart',   type: 'step', description: 'Find the target product' },
    { name: 'subtractQty',         type: 'step', description: 'Reduce quantity by requested amount' },
    { name: 'filterZeroItems',     type: 'step', description: 'Remove items with zero quantity' },
    { name: 'recalculateTotal',    type: 'step', description: 'Recalculate cart total' },
    { name: 'evaluateSuccessType', type: 'step', description: 'Classify the success outcome' },
]

// -- Test data ----------------------------------------------------------------

const sneakersId = 'prod-1' as ProductId
const bookId = 'prod-2' as ProductId
const sneakers: CartItem = { productId: sneakersId, qty: 4 as Quantity, unitPrice: 10000 }
const book: CartItem = { productId: bookId, qty: 1 as Quantity, unitPrice: 1500 }

const cartWith4Sneakers: ActiveCart = {
    status: 'active', id: 'cart-1', customerId: 'c-1',
    items: [sneakers, book],
}

const inputReducing: CoreInput = { cart: cartWith4Sneakers, productId: sneakersId, quantity: 2 as Quantity }
const inputRemoving: CoreInput = { cart: cartWith4Sneakers, productId: bookId, quantity: 1 as Quantity }
const inputEmptying: CoreInput = {
    cart: { ...cartWith4Sneakers, items: [book] },
    productId: bookId, quantity: 1 as Quantity,
}

// Expected outputs...
const expectedReduced: CoreOutput = { ... }
const expectedRemoved: CoreOutput = { ... }
const expectedEmpty: CoreOutput = { status: 'empty', id: 'cart-1' }

// -- Spec ---------------------------------------------------------------------

export const subtractQuantityCoreSpec: Spec<SubtractQuantityCoreFn> = {
    steps,

    shouldFailWith: {
        // Own failures — steps without specs
        'product_not_in_cart': {
            description: 'Should fail when product is not in the cart',
            examples: [
                { description: 'unknown product', whenInput: { cart: cartWith4Sneakers, productId: 'unknown' as ProductId, quantity: 1 as Quantity } },
            ],
        },
        'insufficient_quantity': {
            description: 'Should fail when subtracting more than available',
            examples: [
                { description: 'subtract 10 from 4 in stock', whenInput: { cart: cartWith4Sneakers, productId: sneakersId, quantity: 10 as Quantity } },
            ],
        },
        // Inherited from checkActive: cart_empty, cart_confirmed, cart_cancelled
    },

    shouldSucceedWith: {
        'quantity-reduced': {
            description: 'Should reduce the quantity of the item',
            examples: [{ description: 'reduces 4 sneakers by 2', whenInput: inputReducing, then: expectedReduced }],
        },
        'item-removed': {
            description: 'Should remove the item when quantity reaches zero',
            examples: [{ description: 'removes last book', whenInput: inputRemoving, then: expectedRemoved }],
        },
        'cart-emptied': {
            description: 'Should empty the cart when last item is removed',
            examples: [{ description: 'empties single-item cart', whenInput: inputEmptying, then: expectedEmpty }],
        },
    },

    shouldAssert: {
        'quantity-reduced': {
            'item-still-present': {
                description: 'The product should still be in the cart',
                assert: (input, output) =>
                    output.status === 'active' && output.items.some(i => i.productId === input.productId),
            },
        },
        'item-removed': {
            'item-gone': {
                description: 'The product should no longer be in the cart',
                assert: (input, output) =>
                    output.status === 'active' && !output.items.some(i => i.productId === input.productId),
            },
        },
        'cart-emptied': {
            'status-is-empty': {
                description: 'Cart status should be empty',
                assert: (_input, output) => output.status === 'empty',
            },
        },
    },
}
```

---

## Strategy pattern — spec example

Strategy steps dispatch by data. Each handler has its own spec. The factory
spec just lists the strategy step in the steps array.

```ts
const steps: StepInfo[] = [
    { name: 'validatePayment',     type: 'step',     description: 'Validate payment input',     spec: validatePaymentSpec },
    { name: 'processPayment',      type: 'strategy', description: 'Process by payment type' },
    { name: 'saveResult',          type: 'dep',      description: 'Persist the result' },
    { name: 'evaluateSuccessType', type: 'step',     description: 'Classify the success outcome' },
]
```

The strategy step has `type: 'strategy'` for documentation clarity. It may have
a `spec` if you want to inherit failures from all handlers collectively. More
commonly, handler failures are declared as own failures in the factory's
`shouldFailWith`.

---

## Test file — all functions

Every test file looks the same regardless of function type:

```ts
// check-active.test.ts
import { testSpec } from '../../shared/spec-framework'
import { checkActiveSpec } from './check-active.spec'
import { checkActive } from './check-active'

testSpec('checkActive', checkActiveSpec, checkActive)
```

```ts
// remove-item.test.ts
import { testSpec } from '../../shared/spec-framework'
import { removeItemSpec } from './remove-item.spec'
import { removeItem } from './remove-item'

testSpec('removeItem', removeItemSpec, removeItem)
```

Four lines. Same pattern. The runner handles sync/async, failure inheritance,
assertions — everything.
