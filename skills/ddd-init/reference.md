# DDD Project Conventions

## Folder Structure

Operation-first layout. Each operation gets its own folder. Shell at parent level,
core in `core/` subfolder. Shared steps live in `shared/steps/`.

```
src/
  shared/
    spec-framework.ts       <- Result, SpecFn, Spec, StepInfo, testSpec, inheritFromSteps

  cart/
    types.ts                <- domain types, primitives, failure unions

    shared/steps/           <- reusable atomic steps (used across operations)
      check-active.spec.ts
      check-active.test.ts
      check-active.ts

    subtract-quantity/      <- one folder per operation
      subtract-quantity.spec.ts       <- shell spec
      subtract-quantity.spec.md       <- CLI-generated structural docs (pipeline + decision table)
      subtract-quantity.test.ts
      subtract-quantity.ts            <- shell factory implementation
      core/
        subtract-quantity.spec.ts     <- core spec
        subtract-quantity.spec.md
        subtract-quantity.test.ts
        subtract-quantity.ts          <- core factory implementation

scripts/
  spec-tools.ts             <- flattenSpec, toMarkdownTable, toStepTable
  spec-manifest.ts          <- registry of composed specs
  generate-specs.ts         <- entry point: reads manifest, writes .spec.md
  tsconfig.json

docs/                       <- Jekyll Just the Docs site (business-friendly prose)
  _config.yml               <- Just the Docs theme config
  index.md                  <- Domain home
  cart/
    index.md                <- Aggregate overview (has_children: true)
    subtract-quantity.md    <- Operation page (parent: Cart)
    remove-item.md
    add-item.md

.claude/
  hooks.json                <- PostToolUse hook for .spec.ts auto-regeneration
```

## File Naming

Every function produces up to 4 co-located files:

| File | Purpose | Created by |
|---|---|---|
| `name.spec.ts` | Behavioral contract: SpecFn type + Spec declaration | ddd-spec |
| `name.test.ts` | Test runner wiring (imports + one call) | ddd-test-suite |
| `name.ts` | Implementation | ddd-implement |
| `name.spec.md` | Structural docs: pipeline + decision table (auto-generated) | CLI (`npm run gen:specs`) |

All code files live in the same directory. No separation by concern (no `tests/` folder).

Business-friendly prose docs live separately in `/docs/` as a Jekyll Just the Docs
site, organized by aggregate. Created by the `ddd-documentation` skill.

## Naming Conventions

- **Files:** kebab-case — `check-active.spec.ts`, `subtract-quantity.ts`
- **Exports:** camelCase — `checkActiveSpec`, `subtractQuantityShellSpec`
- **Type exports:** PascalCase — `CheckActiveFn`, `SubtractQuantityShellFn`
- **Failure literals:** snake_case — `cart_empty`, `not_a_string`
- **Success types:** kebab-case, **past tense** — domain events describing what
  happened. `cart-id-parsed`, `quantity-reduced`, `cart-emptied`. Never present
  tense (`cart-is-active`) or noun phrases (`cart-total`)
- **Assertion names:** kebab-case — `status-is-active`, `total-recalculated`

## Core Types

v4 uses two central types instead of three:

### SpecFn — function contract bundle

```ts
type SpecFn<I, O, F extends string, S extends string> = {
    signature: (i: I) => Result<O, F, S>
    asyncSignature: (i: I) => Promise<Result<O, F, S>>
    result: Result<O, F, S>
    input: I
    failures: F
    successTypes: S
    output: O
}
```

Accessed via indexed types: `Fn['signature']`, `Fn['input']`, `Fn['failures']`, etc.

### StrategyFn — strategy dispatch contract

```ts
type StrategyFn<N extends string, I, O, C extends string, F extends string, S extends string> = {
    name: N
    input: I
    output: O
    cases: C
    failures: F
    successTypes: S
    handlers: Record<C, (i: I) => Result<O, F, S>>
}
```

Enforces all handlers share the same input and output types. Accessed via indexed types:
`Fn['handlers']` (for Steps typing), `Fn['failures']`, `Fn['successTypes']`, `Fn['cases']`.

### Spec — behavioral contract

```ts
type Spec<Fn extends AnyFn> = {
    steps?: StepInfo[]
    shouldFailWith: Partial<Record<Fn['failures'], FailGroup<Fn>>>
    shouldSucceedWith: Record<Fn['successTypes'], SuccessGroup<Fn>>
    shouldAssert: Record<Fn['successTypes'], AssertionGroup<Fn>>
}
```

One type for all functions — atomic, core factory, shell factory. The `steps`
array is optional: present for factories, absent for atomic functions.

## Single Input Object

Functions and factories with more than one parameter always take a single object:

```ts
// YES
type CoreInput = { cart: ActiveCart; productId: ProductId; quantity: Quantity }

// NO — separate args
const subtractQuantityCore = (cart: ActiveCart, productId: ProductId, quantity: Quantity) => ...
```

This enables clean spec examples — one `whenInput` field covers the full input.

## Shell / Core Split

- **Shell** — async, has deps (persistence, external services). Lives at the
  operation folder root: `subtract-quantity/subtract-quantity.ts`
- **Core** — pure, sync, no deps. Lives in `core/` subfolder:
  `subtract-quantity/core/subtract-quantity.ts`
- **Shared steps** — pure atomic functions reused across operations. Live in
  `shared/steps/`: `shared/steps/check-active.ts`

Shell calls core as a step. Core composes atomic steps. Atomic steps are leaf nodes.

## Function Taxonomy

```
Shell (async, has deps)
  -> bridges app/infra with domain
  -> parses input (steps), resolves context (deps), calls core (step), persists (deps)
  -> only place async and I/O exist
  -> typed via Fn['asyncSignature']
  -> exported as: factory(steps)(deps) — partial application

Core (sync, pure, no deps)
  -> implements core domain logic of an operation
  -> everything from outside (persistence, context) provided by shell
  -> orchestrates domain steps
  -> typed via Fn['signature']
  -> exported as: factory(steps) — partial application

Step (sync, pure, atomic)
  -> single-concern function (guard, transform, parse)
  -> typed via Fn['signature']
  -> exported directly (no factory)
```

No async domain functions. If it's async, it touches I/O — it's shell.

## Standard Input Shapes

- **Shell input** — `cmd` only (raw, from outside the trust boundary)
  ```ts
  type ShellInput = { cartId: unknown; productId: string; quantity: number }
  ```

- **Core input** — `{ cmd, state, ctx }` (typed domain objects)
  ```ts
  type CoreInput = { cart: ActiveCart; productId: ProductId; quantity: Quantity }
  ```

  The names `cmd`, `state`, `ctx` are conventions — the actual field names
  are domain-specific. Core input bundles the command (what to do), the state
  (current aggregate state), and the context (resolved dependencies).

## Spec Structure

- **Atomic functions** — `Spec<Fn>` without `steps`. Has `shouldFailWith` (all
  failure groups with examples), `shouldSucceedWith`, `shouldAssert`.
- **Factories** — `Spec<Fn>` with `steps: StepInfo[]`. Has `shouldFailWith`
  (partial — only overrides + own failures, rest inherited from step specs),
  `shouldSucceedWith`, `shouldAssert`.

The `testSpec` runner auto-merges inherited failures at runtime via `inheritFromSteps()`.

## Algorithm Visibility via Steps

The `steps` array on a spec serves dual purpose:
1. **Composition** — auto-inherits failures from step specs via `inheritFromSteps()`
2. **Transparency** — makes the algorithm visible alongside the behavioral contract

```ts
const steps: StepInfo[] = [
    { name: 'parseCartId', type: 'step', description: 'Validate and parse the raw cart id', spec: parseCartIdSpec },
    { name: 'findCart',    type: 'dep',  description: 'Fetch cart from persistence by id' },
    { name: 'checkActive', type: 'step', description: 'Verify cart is in active state',    spec: checkActiveSpec },
    { name: 'removeItem',  type: 'step', description: 'Remove the product from cart items' },
    { name: 'saveCart',    type: 'dep',  description: 'Persist the updated cart' },
]
```

### StepInfo — discriminated union

```ts
type StepStep = { name: string; type: 'step'; description: string; spec?: Spec<AnyFn> }
type DepStep  = { name: string; type: 'dep';  description: string }
type StrategyStep = { name: string; type: 'strategy'; description: string; handlers: Record<string, Spec<AnyFn>> }
type StepInfo = StepStep | DepStep | StrategyStep
```

Step types carry only the fields that belong to them. Strategy steps carry `handlers` — a
Record of handler specs keyed by case name — enabling auto-inheritance of handler failures.

Step types:
- `'step'` — pure, sync domain logic. May have a `spec` for failure inheritance.
- `'dep'` — async, I/O. No spec (deps are infrastructure, tested separately).
- `'strategy'` — `Record<Tag, Handler>` dispatch. Carries `handlers` field with handler specs for auto-inheritance.

## Strategy Pattern

When behavior varies by data, use a **strategy step** instead of branching.
A strategy is a `Record<Tag, Handler>` field in `Steps` — the factory dispatches
by property lookup on the input's discriminant. No `if/else`, no `switch`,
no ternary.

**Why it matters:** Factories must stay linear. Conditional statements create
invisible control flow that the spec can't capture. A strategy step makes each
variant a standalone function with its own spec and tests — the factory never
knows which handler runs.

```ts
// ❌ NEVER — conditional in factory body
if (input.coupon.type === 'percentage') {
    result = applyPercentage(input)
} else if (input.coupon.type === 'fixed') {
    result = applyFixed(input)
}

// ✅ ALWAYS — strategy dispatch
const result = steps.calculateDiscount[input.coupon.type](input)
if (!result.ok) return result
```

Each handler is:
- A standalone function in its own file with its own `SpecFn`, `Spec`, and tests
- Typed via `Fn['signature']` like any other step
- Wired into `Steps` as a `Record<Tag, Handler>` field

The factory stays linear — the dispatch is a property access, not a branch.

The strategy contract is typed via `StrategyFn`:

```ts
type DiscountStrategyFn = StrategyFn<
    'calculateDiscount', DiscountInput, DiscountResult, CouponType,
    'rate_out_of_range' | 'discount_exceeds_total' | ...,
    'percentage-applied' | 'fixed-applied' | 'promotion-applied'
>

// Steps type uses the phantom's handlers field
type Steps = { calculateDiscount: DiscountStrategyFn['handlers'] }
```

Handler failures are auto-inherited from the strategy step's `handlers` field via
`inheritFromSteps()`. The factory's `shouldFailWith` can be empty — inherited failures
appear as `test.skip` with `coveredBy: "calculateDiscount (percentage)"` attribution.

## Testing Approach

- **All functions** — tested with `testSpec(name, spec, fn)`. One runner for everything.
- **Test files are minimal** — imports and one `testSpec` call. No logic, no mocks,
  no custom assertions.
- Inherited failures appear as `test.skip` with origin: `"(covered by parseCartId)"`.
- Empty groups without `coveredBy` appear as `test.todo`.

```ts
// Every test file looks like this — 4 lines:
import { testSpec } from '../../shared/spec-framework'
import { checkActiveSpec } from './check-active.spec'
import { checkActive } from './check-active'

testSpec('checkActive', checkActiveSpec, checkActive)
```

## Spec Manifest

Composed specs (factories) are registered in `scripts/spec-manifest.ts`. One entry
per factory. The `ddd-spec` skill adds entries when creating composed specs. Atomic
function specs are not registered — they don't have step trees to flatten.

The manifest drives `npm run gen:specs`, which produces `.spec.md` files with
decision tables. A Claude Code hook auto-runs this when `.spec.ts` files change.

## Implementation Typing

Implementations are typed via the spec's `SpecFn`:

```ts
// Atomic step — typed via Fn['signature']
export const checkActive: CheckActiveFn['signature'] = (cart) => {
    const errors: CheckActiveFn['failures'][] = []
    // ...
}

// Core factory — returns Fn['signature']
const subtractQuantityCoreFactory =
  (steps: CoreSteps): SubtractQuantityCoreFn['signature'] =>
  (input) => { ... }

// Shell factory — returns Fn['asyncSignature']
const subtractQuantityShellFactory =
  (steps: ShellSteps) =>
  (deps: Deps): SubtractQuantityShellFn['asyncSignature'] =>
  async (input) => { ... }
```

Single source of truth — types flow from spec to implementation.
