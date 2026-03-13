# DDD Project Conventions

## Folder Structure

Operation-first layout. Each operation gets its own folder. Shell at parent level,
core in `core/` subfolder. Shared steps live in `shared/steps/`.

```
src/
  shared/
    spec-framework.ts       <- Result, SpecFn, Spec, StepInfo, testSpec, inheritFromSteps, asStepSpec

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
  generate-specs.ts         <- auto-discovers document:true specs, writes .spec.md
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
    document?: boolean
    steps?: StepInfo[]
    shouldFailWith: Partial<Record<Fn['failures'], FailGroup<Fn>>>
    shouldSucceedWith: Record<Fn['successTypes'], SuccessGroup<Fn>>
    shouldAssert: Record<Fn['successTypes'], AssertionGroup<Fn>>
}
```

One type for all functions — atomic, core factory, shell factory. The `steps`
array is optional: present for factories, absent for atomic functions.
`document: true` opts in to `.spec.md` generation via `npm run gen:specs`.

### asStepSpec — AnyFn erasure helper

```ts
const asStepSpec = <Fn extends AnyFn>(spec: Spec<Fn>): Spec<AnyFn> =>
    spec as unknown as Spec<AnyFn>
```

Absorbs the `as unknown as Spec<AnyFn>` cast needed when passing typed specs
to `StepInfo.spec` or `StrategyStep.handlers`. Steps with `never` failures or
any `SpecFn` variant can be used without manual casts:

```ts
// Before — noisy
{ name: 'calculateTotal', type: 'step', spec: calculateTotalSpec as unknown as Spec<AnyFn> }

// After — clean
{ name: 'calculateTotal', type: 'step', spec: asStepSpec(calculateTotalSpec) }
```

### CanonicalFn — standardized implementation structure

```ts
type CanonicalFn<Fn extends AnyFn> = {
    constraints: Record<Fn['failures'], (input: Fn['input']) => boolean>
    conditions:  Record<Fn['successTypes'], (input: Fn['input']) => boolean>
    transform:   Record<Fn['successTypes'], (input: Fn['input']) => Fn['output']>
}
```

An implementation pattern for flat functions (no decomposition needed). The canonical
formula — `constraints → conditions → transform` — standardizes how simple functions
are implemented. `execCanonical(def)` produces a `Fn['signature']` that slots directly
into factory Steps or `testSpec`.

**When to use:** Simple step functions, guard steps, decider-pattern functions.
**When NOT to use:** Factories (they have steps, deps, short-circuiting).

`execCanonical` is an internal implementation detail. The implementation file imports
it, calls it, and exports only the resulting function. Consumers never see
`execCanonical` or the `CanonicalFn` def — they import the function directly:

```ts
// check-active.ts (implementation file)
import { execCanonical } from '../../shared/spec-framework'
const checkActiveDef: CanonicalFn<CheckActiveFn> = { ... }
export const checkActive = execCanonical<CheckActiveFn>(checkActiveDef)

// subtract-quantity/core/subtract-quantity.ts (consumer — sees only the function)
import { checkActive } from '../../shared/steps/check-active'
```

The spec (`Spec<Fn>`) remains separate — `CanonicalFn` is purely an implementation
choice. The same `testSpec(name, spec, fn)` pattern works regardless of whether
the function was implemented manually or via `execCanonical`.

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

- **Shell** — has deps (persistence, external services). Lives at the
  operation folder root: `subtract-quantity/subtract-quantity.ts`
- **Core** — no deps. Lives in `core/` subfolder:
  `subtract-quantity/core/subtract-quantity.ts`
- **Shared steps** — domain functions reused across operations. Live in
  `shared/steps/`: `shared/steps/check-active.ts`

Shell calls core as a step. Core composes domain steps. Steps are leaf nodes.

## Function Taxonomy

The step/dep distinction is about **ownership**, not sync/async:
- **Steps** = domain logic, owned by the factory, baked in at construction
- **Deps** = infrastructure capabilities, injected by the app layer

```
Shell (has deps)
  -> bridges app/infra with domain
  -> parses input (steps), resolves context (deps), calls core (step), persists (deps)
  -> typed via Fn['asyncSignature'] (async because deps are typically async)
  -> exported as: factory(steps)(deps) — partial application
  -> steps are baked in before export; app layer only provides deps

Core (no deps)
  -> implements core domain logic of an operation
  -> everything from outside (persistence, context) provided by shell
  -> orchestrates domain steps
  -> can be sync or async (async if it composes other async domain functions)
  -> typed via Fn['signature'] or Fn['asyncSignature']
  -> exported as: factory(steps) — partial application

Step (atomic, single-concern)
  -> domain function (guard, transform, parse, or composed sub-operation)
  -> can be sync or async — what matters is it's domain logic, not infrastructure
  -> typed via Fn['signature'] or Fn['asyncSignature']
  -> exported directly (no factory) or via factory(steps)
```

**The test for step vs dep:** "Is this domain logic we own, or an infrastructure
capability we need?" If you'd spec it with `Spec<Fn>` and test it with `testSpec`,
it's a step. If it's persistence, an external service, or I/O — it's a dep.

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
- `'step'` — domain logic (sync or async). May have a `spec` for failure inheritance.
- `'dep'` — infrastructure capability (persistence, external service). No spec — tested separately.
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

## Spec Documentation (`.spec.md`)

Specs with `document: true` get auto-generated `.spec.md` files containing
pipeline tables and decision tables. No manual manifest — `npm run gen:specs`
globs for `src/**/*.spec.ts` and processes any spec export with `document: true`.

A Claude Code hook auto-runs this when `.spec.ts` files change.

Typically only factory specs (core or shell) set `document: true` — they have
step trees and decision tables worth generating. Atomic function specs usually don't.

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
