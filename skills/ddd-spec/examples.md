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
import { asStepSpec } from '../../shared/spec-framework'
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
    { name: 'parseCartId', type: 'step', description: 'Validate and parse the raw cart id', spec: asStepSpec(parseCartIdSpec) },
    { name: 'findCart',    type: 'dep',  description: 'Fetch cart from persistence by id' },
    { name: 'checkActive', type: 'step', description: 'Verify cart is in active state',    spec: asStepSpec(checkActiveSpec) },
    { name: 'removeItem',  type: 'step', description: 'Remove the product from cart items' },
    { name: 'saveCart',    type: 'dep',  description: 'Persist the updated cart' },
]

// -- Test data ----------------------------------------------------------------

const CART_ID = '550e8400-e29b-41d4-a716-446655440000'

// -- Spec — behavioral contract + algorithm -----------------------------------

export const removeItemSpec: Spec<RemoveItemFn> = {
    document: true,
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
import { asStepSpec } from '../../../shared/spec-framework'
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
    { name: 'checkActive',         type: 'step', description: 'Verify cart is active',             spec: asStepSpec(checkActiveSpec) },
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
    document: true,
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

## Strategy pattern — complete example

Strategy steps dispatch by a discriminant on the input data. The `StrategyFn`
phantom type enforces that all handlers share the same input and output types —
if a handler has a different shape, it won't compile. Each handler is a
standalone atomic function with its own `SpecFn` and `Spec`. The factory spec
carries a `StrategyStep` with a `handlers` field that maps discriminant values
to handler specs. Handler failures are auto-inherited via `inheritFromSteps` —
no manual `coveredBy` needed.

### StrategyFn — the phantom type contract

```ts
// StrategyFn enforces all handlers share the same input/output types.
// The factory dispatch line `handlers[tag](input)` is only sound when this holds.
export type DiscountStrategyFn = StrategyFn<
    'calculateDiscount', DiscountInput, DiscountResult, CouponType,
    'rate_out_of_range' | 'discount_exceeds_total' | 'product_not_in_cart' | 'insufficient_items_for_promotion',
    'percentage-applied' | 'fixed-applied' | 'promotion-applied'
>
```

### Handler specs — one per variant

Each handler is an atomic function spec. It gets its own `SpecFn`, `Spec`,
examples, and assertions. All three handlers below take `DiscountInput` and
return `DiscountResult` — enforced by the `StrategyFn` above.

```ts
// apply-percentage.spec.ts
import type { SpecFn, Spec } from '../spec-framework'
import type { DiscountInput, DiscountResult, ActiveCart } from '../types'

export type ApplyPercentageFn = SpecFn<
    DiscountInput,
    DiscountResult,
    'rate_out_of_range',
    'percentage-applied'
>

const cart: ActiveCart = {
    status: 'active', id: 'cart-1', customerId: 'c-1',
    items: [
        { productId: 'shoes-1', qty: 2, unitPrice: 5000 },  // 2 × $50 = $100
        { productId: 'socks-1', qty: 3, unitPrice: 1000 },  // 3 × $10 = $30
    ],
    // Total: 13000
}

export const applyPercentageSpec: Spec<ApplyPercentageFn> = {
    shouldFailWith: {
        'rate_out_of_range': {
            description: 'Percentage rate must be between 1 and 100 inclusive',
            examples: [
                { description: 'zero rate is rejected',      whenInput: { cart, coupon: { type: 'percentage', rate: 0 } } },
                { description: 'negative rate is rejected',  whenInput: { cart, coupon: { type: 'percentage', rate: -10 } } },
                { description: 'rate above 100 is rejected', whenInput: { cart, coupon: { type: 'percentage', rate: 101 } } },
            ],
        },
    },
    shouldSucceedWith: {
        'percentage-applied': {
            description: 'Discount calculated as percentage of cart total',
            examples: [
                {
                    description: '20% off a $130 cart saves $26',
                    whenInput: { cart, coupon: { type: 'percentage', rate: 20 } },
                    then: { originalTotal: 13000, savedAmount: 2600, finalTotal: 10400 },
                },
                {
                    description: '100% off makes cart free (boundary)',
                    whenInput: { cart, coupon: { type: 'percentage', rate: 100 } },
                    then: { originalTotal: 13000, savedAmount: 13000, finalTotal: 0 },
                },
                {
                    description: '1% off minimum discount (boundary)',
                    whenInput: { cart, coupon: { type: 'percentage', rate: 1 } },
                    then: { originalTotal: 13000, savedAmount: 130, finalTotal: 12870 },
                },
            ],
        },
    },
    shouldAssert: {
        'percentage-applied': {
            'saved-matches-rate': {
                description: 'Saved amount equals original total times rate / 100',
                assert: (input, output) =>
                    output.savedAmount === Math.floor(output.originalTotal * (input.coupon as { rate: number }).rate / 100),
            },
            'total-is-consistent': {
                description: 'Final total equals original minus saved',
                assert: (_input, output) =>
                    output.finalTotal === output.originalTotal - output.savedAmount,
            },
        },
    },
}
```

```ts
// apply-fixed.spec.ts
import type { SpecFn, Spec } from '../spec-framework'
import type { DiscountInput, DiscountResult, ActiveCart } from '../types'

export type ApplyFixedFn = SpecFn<
    DiscountInput,
    DiscountResult,
    'discount_exceeds_total',
    'fixed-applied'
>

const cart: ActiveCart = {
    status: 'active', id: 'cart-1', customerId: 'c-1',
    items: [
        { productId: 'shoes-1', qty: 2, unitPrice: 5000 },
        { productId: 'socks-1', qty: 3, unitPrice: 1000 },
    ],
    // Total: 13000
}

export const applyFixedSpec: Spec<ApplyFixedFn> = {
    shouldFailWith: {
        'discount_exceeds_total': {
            description: 'Fixed discount amount must not exceed the cart total',
            examples: [
                { description: '$150 off a $130 cart is rejected', whenInput: { cart, coupon: { type: 'fixed', amount: 15000 } } },
                { description: 'one cent over total is rejected',  whenInput: { cart, coupon: { type: 'fixed', amount: 13001 } } },
            ],
        },
    },
    shouldSucceedWith: {
        'fixed-applied': {
            description: 'Fixed amount subtracted from cart total',
            examples: [
                {
                    description: '$25 off a $130 cart',
                    whenInput: { cart, coupon: { type: 'fixed', amount: 2500 } },
                    then: { originalTotal: 13000, savedAmount: 2500, finalTotal: 10500 },
                },
                {
                    description: 'exact total makes cart free (boundary)',
                    whenInput: { cart, coupon: { type: 'fixed', amount: 13000 } },
                    then: { originalTotal: 13000, savedAmount: 13000, finalTotal: 0 },
                },
            ],
        },
    },
    shouldAssert: {
        'fixed-applied': {
            'saved-matches-amount': {
                description: 'Saved amount equals the coupon amount',
                assert: (input, output) =>
                    output.savedAmount === (input.coupon as { amount: number }).amount,
            },
            'total-is-consistent': {
                description: 'Final total equals original minus saved',
                assert: (_input, output) =>
                    output.finalTotal === output.originalTotal - output.savedAmount,
            },
        },
    },
}
```

```ts
// apply-buy-x-get-y.spec.ts
import type { SpecFn, Spec } from '../spec-framework'
import type { DiscountInput, DiscountResult, ActiveCart } from '../types'

export type ApplyBuyXGetYFn = SpecFn<
    DiscountInput,
    DiscountResult,
    'product_not_in_cart' | 'insufficient_items_for_promotion',
    'promotion-applied'
>

const cart: ActiveCart = {
    status: 'active', id: 'cart-1', customerId: 'c-1',
    items: [
        { productId: 'shoes-1', qty: 2, unitPrice: 5000 },  // 2 × $50
        { productId: 'socks-1', qty: 3, unitPrice: 1000 },  // 3 × $10
    ],
    // Total: 13000
}

export const applyBuyXGetYSpec: Spec<ApplyBuyXGetYFn> = {
    shouldFailWith: {
        'product_not_in_cart': {
            description: 'The promoted product must exist in the cart',
            examples: [
                {
                    description: 'promotion on hat-1 but cart has no hats',
                    whenInput: {
                        cart,
                        coupon: { type: 'buy-x-get-y', productId: 'hat-1', buyQty: 1, freeQty: 1 },
                    },
                },
            ],
        },
        'insufficient_items_for_promotion': {
            description: 'Cart must have at least buyQty of the promoted product',
            examples: [
                {
                    description: 'buy 5 get 1 free but only 2 shoes in cart',
                    whenInput: {
                        cart,
                        coupon: { type: 'buy-x-get-y', productId: 'shoes-1', buyQty: 5, freeQty: 1 },
                    },
                },
            ],
        },
    },
    shouldSucceedWith: {
        'promotion-applied': {
            description: 'Free items deducted from cart total',
            examples: [
                {
                    description: 'buy 2 shoes get 1 free saves $50',
                    whenInput: {
                        cart,
                        coupon: { type: 'buy-x-get-y', productId: 'shoes-1', buyQty: 2, freeQty: 1 },
                    },
                    then: { originalTotal: 13000, savedAmount: 5000, finalTotal: 8000 },
                },
                {
                    description: 'buy 3 socks get 2 free saves $20',
                    whenInput: {
                        cart,
                        coupon: { type: 'buy-x-get-y', productId: 'socks-1', buyQty: 3, freeQty: 2 },
                    },
                    then: { originalTotal: 13000, savedAmount: 2000, finalTotal: 11000 },
                },
            ],
        },
    },
    shouldAssert: {
        'promotion-applied': {
            'saved-matches-free-items': {
                description: 'Saved amount equals freeQty times unit price of promoted product',
                assert: (input, output) => {
                    const coupon = input.coupon as { productId: string; freeQty: number }
                    const item = input.cart.items.find(i => i.productId === coupon.productId)!
                    return output.savedAmount === Math.min(coupon.freeQty, item.qty) * item.unitPrice
                },
            },
            'total-is-consistent': {
                description: 'Final total equals original minus saved',
                assert: (_input, output) =>
                    output.finalTotal === output.originalTotal - output.savedAmount,
            },
        },
    },
}
```

### Factory spec with strategy step

The factory declares a `StrategyStep` in the `steps` array. The step carries a
`handlers` field mapping each discriminant value to its handler spec. The
`ApplyDiscountFn` type references `DiscountStrategyFn['failures']` and
`DiscountStrategyFn['successTypes']` to stay in sync with the phantom type.

`shouldFailWith` is completely empty — all handler failures are auto-inherited
via `inheritFromSteps`. At test time, `testSpec()` walks the strategy step's
handlers, discovers their failures, and emits `test.skip` entries with
attribution like `"rate_out_of_range — ... (covered by calculateDiscount
(percentage))"`.

```ts
// apply-discount.spec.ts — factory with strategy step
import type { SpecFn, StrategyFn, Spec, StepInfo } from '../spec-framework'
import type { DiscountInput, DiscountResult, ActiveCart, CouponType } from '../types'
import { applyPercentageSpec } from './apply-percentage.spec'
import { applyFixedSpec } from './apply-fixed.spec'
import { applyBuyXGetYSpec } from './apply-buy-x-get-y.spec'

// -- Strategy contract --------------------------------------------------------
// Enforces: all handlers share the same input/output types.
// Without this, plugging a handler with a different input shape would compile
// but break at runtime when the factory dispatches.

export type DiscountStrategyFn = StrategyFn<
    'calculateDiscount',
    DiscountInput,
    DiscountResult,
    CouponType,
    // All handler failures aggregated
    | 'rate_out_of_range'
    | 'discount_exceeds_total'
    | 'product_not_in_cart'
    | 'insufficient_items_for_promotion',
    // All handler success types aggregated
    'percentage-applied' | 'fixed-applied' | 'promotion-applied'
>

// -- Function contract --------------------------------------------------------

export type ApplyDiscountFn = SpecFn<
    DiscountInput,
    DiscountResult,
    DiscountStrategyFn['failures'],
    DiscountStrategyFn['successTypes']
>

// -- Algorithm ----------------------------------------------------------------

const steps: StepInfo[] = [
    {
        name: 'calculateDiscount',
        type: 'strategy',
        description: 'Calculate discount by coupon type (percentage / fixed / buy-x-get-y)',
        handlers: {
            percentage:     applyPercentageSpec,
            fixed:          applyFixedSpec,
            'buy-x-get-y':  applyBuyXGetYSpec,
        },
    },
]

// -- Test data ----------------------------------------------------------------

const cart: ActiveCart = {
    status: 'active', id: 'cart-1', customerId: 'c-1',
    items: [
        { productId: 'shoes-1', qty: 2, unitPrice: 5000 },
        { productId: 'socks-1', qty: 3, unitPrice: 1000 },
    ],
    // Total: 13000
}

// -- Spec — behavioral contract -----------------------------------------------
// shouldFailWith is empty — all handler failures auto-inherited via inheritFromSteps.

export const applyDiscountSpec: Spec<ApplyDiscountFn> = {
    document: true,
    steps,

    shouldFailWith: {
        // All handler failures are inherited automatically from the strategy step's handlers.
        // They appear as test.skip with attribution:
        //   "rate_out_of_range — ... (covered by calculateDiscount (percentage))"
        //   "discount_exceeds_total — ... (covered by calculateDiscount (fixed))"
        //   etc.
    },

    shouldSucceedWith: {
        'percentage-applied': {
            description: 'Percentage discount calculated from cart total',
            examples: [
                {
                    description: '20% off via percentage coupon',
                    whenInput: { cart, coupon: { type: 'percentage', rate: 20 } },
                    then: { originalTotal: 13000, savedAmount: 2600, finalTotal: 10400 },
                },
            ],
        },
        'fixed-applied': {
            description: 'Fixed amount subtracted from cart total',
            examples: [
                {
                    description: '$25 off via fixed coupon',
                    whenInput: { cart, coupon: { type: 'fixed', amount: 2500 } },
                    then: { originalTotal: 13000, savedAmount: 2500, finalTotal: 10500 },
                },
            ],
        },
        'promotion-applied': {
            description: 'Free items deducted via buy-x-get-y promotion',
            examples: [
                {
                    description: 'buy 2 shoes get 1 free via promotion coupon',
                    whenInput: {
                        cart,
                        coupon: { type: 'buy-x-get-y', productId: 'shoes-1', buyQty: 2, freeQty: 1 },
                    },
                    then: { originalTotal: 13000, savedAmount: 5000, finalTotal: 8000 },
                },
            ],
        },
    },

    shouldAssert: {
        'percentage-applied': {
            'total-is-consistent': {
                description: 'Final total equals original minus saved',
                assert: (_input, output) =>
                    output.finalTotal === output.originalTotal - output.savedAmount,
            },
        },
        'fixed-applied': {
            'total-is-consistent': {
                description: 'Final total equals original minus saved',
                assert: (_input, output) =>
                    output.finalTotal === output.originalTotal - output.savedAmount,
            },
        },
        'promotion-applied': {
            'total-is-consistent': {
                description: 'Final total equals original minus saved',
                assert: (_input, output) =>
                    output.finalTotal === output.originalTotal - output.savedAmount,
            },
        },
    },
}
```

Key differences from composed factory specs:

- **`StrategyFn` phantom type** — enforces all handlers share `DiscountInput`
  and `DiscountResult`. Without it, a handler with a different input shape would
  compile but break at runtime when the factory dispatches by tag.
- **`StrategyStep` with `handlers` field** — maps each discriminant value
  (`percentage`, `fixed`, `buy-x-get-y`) to its handler spec. The test runner
  walks this map to discover and attribute inherited failures.
- **`shouldFailWith: {}`** — completely empty. All four handler failures
  (`rate_out_of_range`, `discount_exceeds_total`, `product_not_in_cart`,
  `insufficient_items_for_promotion`) are auto-inherited from the strategy
  step's handler specs via `inheritFromSteps`. No manual `coveredBy` needed.
- **`ApplyDiscountFn` references the phantom type** —
  `DiscountStrategyFn['failures']` and `DiscountStrategyFn['successTypes']`
  keep the factory's type parameters in sync with the strategy contract.
- **One success example per handler outcome** — the factory proves each
  dispatch path works end-to-end. Detailed edge cases live in the handler specs.

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
