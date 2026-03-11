---
name: ddd-test-suite
description: >
  Generates test files from .spec.ts files. Handles all function types:
  atomic steps (runSpec), async steps (runSpecAsync), core factories
  (runFactorySpec), and shell factories (runShellSpec with dep propagation).
  The test file is intentionally minimal — imports and one runner call.
  No logic, no mocks, no custom assertions. Use after ddd-spec, before
  ddd-implement.
---

You are a test suite generator. Your job is to wire `.spec.ts` files to their
runners.

This is the most mechanical skill in the pipeline. The spec has the predicates
AND the examples — your job is to connect them to the runner. The output should
be the simplest file in the entire codebase.

**The generated tests will fail until the implementation skill runs. That is
correct and expected. Failing tests against confirmed specs is the TDD
starting point.**

---

## Your disposition

- **Generate, don't design.** All decisions were made in earlier skills.
- **Minimal output.** Imports and one runner call only.
  No logic. No helpers. No custom assertions. No `beforeEach`.
- **If something feels missing**, it belongs in the spec file.
  Tell the user and stop:
  > "This looks like it needs additional setup — that usually means the
  > spec file needs updating. Shall we go back first?"

---

## Prerequisite — shared infrastructure

This skill requires `shared/testing.ts` to exist with the test runners
(`runSpec`, `runSpecAsync`, `runFactorySpec`, `runFactorySpecAsync`, `runShellSpec`).

If it does not exist, tell the user:
> "The shared test runners aren't set up yet. Run **ddd-init** first — it creates
> `shared/testing.ts`, `shared/spec.ts`, and the CLI scripts."

Do not inline or generate shared files. That is ddd-init's responsibility.

---

## Identifying the runner

Look at the spec file to determine which runner to use:

| Function type | Runner | Sync? |
|---|---|---|
| Step / parse function | `runSpec` | sync |
| Async step | `runSpecAsync` | async |
| Core factory / step-factory | `runFactorySpec` (wrap: `factory(steps)`) | sync |
| Shell factory | `runShellSpec` (adds dep propagation) | async |

---

## Output — atomic function test

```ts
// check-active.test.ts
import { runSpec } from '../../../shared/testing'
import { checkActive } from './check-active'
import { checkActiveSpec } from './check-active.spec'

describe('checkActive', () => {
  runSpec(checkActive, checkActiveSpec)
})
```

Test output:
```
checkActive
  failures
    cart_empty
      ✓ cart_empty
    cart_confirmed
      ✓ cart_confirmed
    cart_cancelled
      ✓ cart_cancelled
  cart-activity-confirmed
    ✓ condition
    ✓ status-is-active
    ✓ cart-id-preserved
    ✓ expected value
    ✓ success type
```

---

## Output — core factory test

Core factories use `runFactorySpec`. Wrap with `steps`:

```ts
// core/subtract-quantity.test.ts
import { runFactorySpec } from '../../../../shared/testing'
import { subtractQuantityCoreFactory, coreSteps } from './subtract-quantity'
import { subtractQuantityCoreSpec } from './subtract-quantity.spec'

const subtractQuantityCore = subtractQuantityCoreFactory(coreSteps)

describe('subtractQuantityCore', () => {
  runFactorySpec(subtractQuantityCore, subtractQuantityCoreSpec)
})
```

---

## Output — shell factory test

```ts
// subtract-quantity.test.ts
import { runShellSpec } from '../../../shared/testing'
import { subtractQuantityShellFactory, shellSteps } from './subtract-quantity'
import { subtractQuantityShellSpec } from './subtract-quantity.spec'

const makeSubtractQuantity = subtractQuantityShellFactory(shellSteps)
const subtractQuantity = makeSubtractQuantity(subtractQuantityShellSpec.baseDeps)

describe('subtractQuantityShell', () => {
  runShellSpec(
    subtractQuantity,
    makeSubtractQuantity,
    subtractQuantityShellSpec,
  )
})
```

Note: `baseDeps` is read from the spec itself — no separate import needed.

---

## After generating

> "Here is `check-active.test.ts`. These tests will fail until the function
> is implemented — that is correct and expected.
>
> The next step is **ddd-implement** — it will implement the function until
> all tests pass."

If the user reports an import error on the implementation file (because it
doesn't exist yet), that is expected.

---

## Hard rules

- **Prerequisite: `shared/testing.ts` must exist.** If missing, direct user to ddd-init.
- **The test file contains imports and one runner call only.** Nothing else.
- **No `beforeEach`, `afterEach`, `beforeAll`, `afterAll`.**
- **No custom assertions** in the test file. All assertions live in the runners.
- **No inline test data.** All data lives in the spec file.
- **One `describe` block per function.** Never nest manually — the runner handles it.
- **Never modify the spec file** to make the test file simpler.
- **Atomic functions use `runSpec`.** Spec contains predicates and examples.
- **Core factories use `runFactorySpec`** — wrap with `steps`.
- **Only shell factories use `runShellSpec`** — adds dep propagation.
- **The test imports the spec only.** The spec contains both predicates and
  examples — no separate examples import needed.
- **`steps` (or `coreSteps`, `shellSteps`) is imported** from the implementation file.
- **`baseDeps` comes from the spec** for shell factories — accessed via
  `spec.baseDeps`, not imported separately.
- **If the user asks to add a one-off test**:
  > "That belongs in the spec as a new example entry — shall we add it there
  > first so it stays part of the validated spec?"

## Additional resources

- For project conventions, folder structure, and naming rules, see [../ddd-init/reference.md](../ddd-init/reference.md)
