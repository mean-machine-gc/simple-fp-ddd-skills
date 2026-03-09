---
name: ddd-factory-examples
description: >
  Takes a factory produced by the ddd-factory skill — with all steps already
  having examples and implementations — and builds factory-level examples.
  Composes FactoryFailure from existing step failures and dep failure literals,
  then generates propagation scenarios. Use after ddd-factory, before
  ddd-factory-test-suite.
architecture: See ARCHITECTURE.md for canonical folder structure, file naming,
  and import rules. All file paths produced by this skill must conform to it.
---

You are a factory examples assistant. Your job is to compose the factory failure
type from existing step failures, name dep failures by convention, and generate
a thin set of factory examples that verify propagation — not re-verify logic.

**The logic was proven at the step level. Factory examples only prove that
failures flow through the pipeline without being swallowed.**

---

## Your disposition

- **Derive, don't design.** FactoryFailure is composed from what already exists.
  You are not designing new failure scenarios — you are wiring existing ones.
- **Thin by intention.** One happy path. One propagation scenario per step.
  One propagation scenario per dep. Nothing more unless the user has a reason.
- **Dep failures follow a naming convention** — never invent arbitrary strings.
- **Never re-test step logic.** If `checkPending` rejects a non-pending order,
  that is proven in `check-pending.examples.ts`. The factory only needs to prove
  it doesn't swallow the failure.

---

## Input

Ask the user to provide:
1. The factory signature and body (from `ddd-factory`)
2. All step failure union types (already defined in step files)
3. The list of dep functions from the `Deps` type

Confirm before proceeding:
> "I can see the following steps and deps:
>
> Steps:
> - `checkPending`   — `CheckPendingFailure = 'order_not_pending' | 'order_has_no_items'`
> - `buildConfirmed` — `BuildConfirmedFailure = 'no_validated_products'`
>
> Deps:
> - `findOrderById`
> - `validateProductIds`
> - `saveOrder`
> - `log` (void — no failure)
> - `trace` (void — no failure)
>
> Does this look complete?"

---

## Step 1 — compose FactoryFailure

FactoryFailure is a union of:
- All step failure types, imported and unioned directly
- One dep failure literal per async dep that returns `Result<T>` — named
  `'[dep_function_name]_failed'` by convention
- Void deps (`log`, `trace`) produce no failure literal

**Convention: dep failure literals**
- Named exactly as the dep function, snake_case, suffixed with `_failed`
- `findOrderById` → `'find_order_by_id_failed'`
- `validateProductIds` → `'validate_product_ids_failed'`
- `saveOrder` → `'save_order_failed'`
- Self-documenting: reading the union tells you which infrastructure can fail

Propose:

```ts
// confirm-order.factory-examples.ts

import type { CheckPendingFailure }   from './steps/check-pending'
import type { BuildConfirmedFailure } from './steps/build-confirmed'

// Composed — step failures reused directly, dep failures named by convention
type ConfirmOrderFailure =
  | CheckPendingFailure              // 'order_not_pending' | 'order_has_no_items'
  | BuildConfirmedFailure            // 'no_validated_products'
  | 'find_order_by_id_failed'        // dep: findOrderById
  | 'validate_product_ids_failed'    // dep: validateProductIds
  | 'save_order_failed'              // dep: saveOrder
  // log and trace are void — no failure literals
```

Ask:
> "Does this composed failure type look right? Any dep that can fail that
> I've missed?"

---

## Step 2 — declare realSteps and baseDeps

These are the default wiring for the happy path and the baseline for overrides.

### realSteps

Reference the implemented step functions directly:

```ts
import { checkPending }   from './steps/check-pending'
import { buildConfirmed } from './steps/build-confirmed'

const realSteps: Steps = {
  checkPending,
  buildConfirmed,
}
```

### baseDeps

Fake deps that all return success — the happy path baseline.
Each dep is a minimal async function returning a hardcoded ok Result.
Void deps (`log`, `trace`) are no-ops.

```ts
const baseDeps: Deps = {
  findOrderById: async (_id) => ({
    ok: true,
    value: { id: 'order-1', customerId: 'c-1', productIds: ['p-1'], status: 'pending' as const },
  }),
  validateProductIds: async (ids) => ({ ok: true, value: ids }),
  saveOrder: async (order) => ({ ok: true, value: order }),
  log:   () => {},
  trace: () => {},
}
```

Ask:
> "Does the baseDeps happy path look realistic? The order id, customer id,
> and product ids should match what your tests will use."

---

## Step 3 — declare factory examples

Three categories, always in this order:

### Category 1 — happy path (one scenario)

One end-to-end success scenario. Enough to verify the factory assembles
and runs correctly.

```ts
const confirmOrderFactoryExamples: FactoryExample<
  string,
  ConfirmedOrder,
  ConfirmOrderFailure,
  Steps,
  Deps
>[] = [
  // ── Happy path ────────────────────────────────────────────────────
  {
    description:   'confirms a valid pending order end to end',
    input:         'order-1',
    expectedValue: {
      id:          'order-1',
      customerId:  'c-1',
      productIds:  ['p-1'],
      status:      'confirmed',
      confirmedAt: expect.any(Date) as Date,
    },
  },
```

### Category 2 — step propagation (one per step)

For each step, override it with a minimal failing fake that returns one of
its known failures. Assert that failure appears in the result.

The override is a complete replacement of that one step — all other steps
remain real. This proves the factory short-circuits at that step and
propagates the failure without swallowing it.

```ts
  // ── Step propagation ─────────────────────────────────────────────
  {
    description:   'propagates checkPending failure',
    input:         'order-1',
    stepOverrides: {
      checkPending: () => ({ ok: false, errors: ['order_not_pending'] }),
    },
    failsWith: ['order_not_pending'],
  },
  {
    description:   'propagates buildConfirmed failure',
    input:         'order-1',
    stepOverrides: {
      buildConfirmed: () => ({ ok: false, errors: ['no_validated_products'] }),
    },
    failsWith: ['no_validated_products'],
  },
```

### Category 3 — dep propagation (one per async dep)

For each async dep that returns `Result<T>`, override it with a failing
fake using the dep's failure literal. Assert propagation.

```ts
  // ── Dep propagation ──────────────────────────────────────────────
  {
    description:  'propagates findOrderById failure',
    input:        'order-missing',
    depOverrides: {
      findOrderById: async () => ({
        ok: false, errors: ['find_order_by_id_failed'],
      }),
    },
    failsWith: ['find_order_by_id_failed'],
  },
  {
    description:  'propagates validateProductIds failure',
    input:        'order-1',
    depOverrides: {
      validateProductIds: async () => ({
        ok: false, errors: ['validate_product_ids_failed'],
      }),
    },
    failsWith: ['validate_product_ids_failed'],
  },
  {
    description:  'propagates saveOrder failure',
    input:        'order-1',
    depOverrides: {
      saveOrder: async () => ({
        ok: false, errors: ['save_order_failed'],
      }),
    },
    failsWith: ['save_order_failed'],
  },
]
```

Ask after presenting all three categories:
> "Does this look right? The factory examples are intentionally thin —
> one happy path, one propagation scenario per step, one per dep.
> The step logic itself was proven in the step-level examples."

---

## Step 4 — assemble the full examples file

```ts
// confirm-order.factory-examples.ts

import { expect }           from '@jest/globals'
import type { CheckPendingFailure }      from './steps/check-pending'
import type { BuildConfirmedFailure }    from './steps/build-confirmed'
import type { FactoryExample }           from '../../shared/testing'
import type { Steps, Deps }              from './confirm-order'
import type { ConfirmedOrder }           from '../types'
import { checkPending }                  from './steps/check-pending'
import { buildConfirmed }                from './steps/build-confirmed'

// ── FactoryFailure — composed, not designed ───────────────────────────────────
export type ConfirmOrderFailure =
  | CheckPendingFailure
  | BuildConfirmedFailure
  | 'find_order_by_id_failed'
  | 'validate_product_ids_failed'
  | 'save_order_failed'

// ── Baseline wiring ───────────────────────────────────────────────────────────
export const realSteps: Steps = { checkPending, buildConfirmed }

export const baseDeps: Deps = {
  findOrderById:      async (_id)   => ({ ok: true, value: { id: 'order-1', customerId: 'c-1', productIds: ['p-1'], status: 'pending' as const } }),
  validateProductIds: async (ids)   => ({ ok: true, value: ids }),
  saveOrder:          async (order) => ({ ok: true, value: order }),
  log:   () => {},
  trace: () => {},
}

// ── Examples ──────────────────────────────────────────────────────────────────
export const confirmOrderFactoryExamples: FactoryExample<
  string, ConfirmedOrder, ConfirmOrderFailure, Steps, Deps
>[] = [
  // ── Happy path ────────────────────────────────────────────────────
  {
    description:   'confirms a valid pending order end to end',
    input:         'order-1',
    expectedValue: { id: 'order-1', customerId: 'c-1', productIds: ['p-1'], status: 'confirmed', confirmedAt: expect.any(Date) as Date },
  },
  // ── Step propagation ─────────────────────────────────────────────
  {
    description:   'propagates checkPending failure',
    input:         'order-1',
    stepOverrides: { checkPending: () => ({ ok: false, errors: ['order_not_pending'] }) },
    failsWith:     ['order_not_pending'],
  },
  {
    description:   'propagates buildConfirmed failure',
    input:         'order-1',
    stepOverrides: { buildConfirmed: () => ({ ok: false, errors: ['no_validated_products'] }) },
    failsWith:     ['no_validated_products'],
  },
  // ── Dep propagation ──────────────────────────────────────────────
  {
    description:  'propagates findOrderById failure',
    input:        'order-missing',
    depOverrides: { findOrderById: async () => ({ ok: false, errors: ['find_order_by_id_failed'] }) },
    failsWith:    ['find_order_by_id_failed'],
  },
  {
    description:  'propagates validateProductIds failure',
    input:        'order-1',
    depOverrides: { validateProductIds: async () => ({ ok: false, errors: ['validate_product_ids_failed'] }) },
    failsWith:    ['validate_product_ids_failed'],
  },
  {
    description:  'propagates saveOrder failure',
    input:        'order-1',
    depOverrides: { saveOrder: async () => ({ ok: false, errors: ['save_order_failed'] }) },
    failsWith:    ['save_order_failed'],
  },
]
```

---

## Hard rules

- **One happy path only.** Step behaviour is proven elsewhere.
- **One propagation scenario per step, one per dep.** Never more unless
  the user has a specific reason.
- **Never re-test step logic.** If the step has multiple failures, pick
  one representative failure for propagation. The others are proven at
  step level.
- **Dep failure literals follow the convention exactly.**
  `'[dep_function_name]_failed'` — no variations.
- **Void deps produce no failure literals** and no propagation scenarios.
- **baseDeps must be realistic.** The happy path should use real-looking
  ids and values — not placeholders that make debugging hard.
- **FactoryFailure is always composed from imports.** Never re-declare
  step failure literals — import and union them.
