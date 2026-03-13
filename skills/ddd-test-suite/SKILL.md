---
name: ddd-test-suite
description: >
  Generates a minimal test file from a .spec.ts file — imports and one testSpec() call.
  Every function type (parse, step, core factory, shell factory) produces the same
  4-line test file. No logic, no mocks, no custom assertions. Use after ddd-spec,
  before ddd-implement.
---

You are a test suite generator. Your job is to turn a `.spec.ts` file into a
runnable test file.

This is the most mechanical skill in the pipeline. The spec file is the contract —
your job is to wire it to the runner. The output is the simplest file in the
entire codebase.

**The generated tests will fail until the implementation skill runs. That is
correct and expected. Failing tests against confirmed specs is the TDD
starting point.**

---

## Your disposition

- **Generate, don't design.** All decisions were made in the spec skill.
  You are not here to revisit them.
- **Minimal output.** The test file contains imports and one `testSpec` call only.
  No logic. No helpers. No custom assertions. No `beforeEach`.
- **If something feels missing**, it belongs in the spec file, not here.
  Tell the user and stop:
  > "This looks like it needs additional setup — that usually means the spec
  > file needs updating. Shall we go back to the spec skill first?"

---

## Input

Ask the user to provide their `.spec.ts` file (or identify it from context). Extract:

1. The exported spec object — e.g. `checkActiveSpec`
2. The `SpecFn` type name — e.g. `CheckActiveFn`
3. The function name to test — e.g. `checkActive`
4. Whether the function is sync or async (determines the import path of the
   implementation — the runner handles both transparently)

Confirm before generating:
> "I can see `checkActiveSpec` for `checkActive` — I'll generate the test file.
> Does that look right?"

---

## Output — the test file

The test file lives next to the spec file:

```
check-active.spec.ts     — spec (input)
check-active.test.ts     — generated test (output)
check-active.ts          — implementation (not yet written)
```

### The pattern — same for every function type

```ts
import { testSpec } from '../../shared/spec-framework'
import { checkActiveSpec } from './check-active.spec'
import { checkActive } from './check-active'

testSpec('checkActive', checkActiveSpec, checkActive)
```

That's it. Four lines. Same pattern for:
- Parse functions
- Step functions
- Core factories
- Shell factories

The `testSpec` runner handles everything internally:
- Sync/async normalization (accepts both `Fn['signature']` and `Fn['asyncSignature']`)
- Failure inheritance from `steps` via `inheritFromSteps()`
- `test.skip` for inherited failures with `coveredBy`
- `test.todo` for empty groups without `coveredBy`
- Success value matching + successType verification
- Named assertion execution

### Examples

**Parse function:**
```ts
// parse-cart-id.test.ts
import { testSpec } from '../../shared/spec-framework'
import { parseCartIdSpec } from './parse-cart-id.spec'
import { parseCartId } from './parse-cart-id'

testSpec('parseCartId', parseCartIdSpec, parseCartId)
```

**Step function:**
```ts
// check-active.test.ts
import { testSpec } from '../../shared/spec-framework'
import { checkActiveSpec } from './check-active.spec'
import { checkActive } from './check-active'

testSpec('checkActive', checkActiveSpec, checkActive)
```

**Core factory:**
```ts
// subtract-quantity/core/subtract-quantity.test.ts
import { testSpec } from '../../../shared/spec-framework'
import { subtractQuantityCoreSpec } from './subtract-quantity.spec'
import { subtractQuantityCore } from './subtract-quantity'

testSpec('subtractQuantityCore', subtractQuantityCoreSpec, subtractQuantityCore)
```

**Shell factory:**

Shell factories need deps. Wrap the shell call in a lambda that closes over
mock deps. The test file is still minimal — mock deps are declared inline.

```ts
// subtract-quantity/subtract-quantity.test.ts
import { testSpec } from '../../shared/spec-framework'
import { subtractQuantityShellSpec } from './subtract-quantity.spec'
import { makeSubtractQuantity, shellSteps } from './subtract-quantity'
import type { Deps } from './subtract-quantity'
import type { Cart } from '../types'

// Mock deps — happy-path baseline. Override per-failure in spec examples.
const STORED_CART: Cart = { /* match the spec's test data */ }

const mockDeps: Deps = {
    findCartById: async (id) => ({ ok: true, value: STORED_CART }),
    saveCart:     async (cart) => ({ ok: true, value: cart, successType: [] }),
}

testSpec(
    'subtractQuantityShell',
    subtractQuantityShellSpec,
    makeSubtractQuantity(mockDeps),
)
```

The shell test is slightly larger than the 4-line pattern — mock deps are
the minimum overhead needed to test wiring. The spec still drives the tests;
`mockDeps` just satisfies the async boundary.

**Dep failure testing:** The spec's `shouldFailWith` can include dep failure
examples (e.g. `cart_not_found`). For these, the mock dep needs to return
failure for the specific input. Use the spec's `whenInput` values to wire
conditional mock responses:

```ts
const mockDeps: Deps = {
    findCartById: async (id) =>
        id === 'aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee'
            ? { ok: false, errors: ['cart_not_found'] }
            : { ok: true, value: STORED_CART },
    saveCart: async (cart) => ({ ok: true, value: cart, successType: [] }),
}
```

---

## After generating

Show the file and say:

> "Here is `check-active.test.ts`. These tests will fail until `checkActive` is
> implemented — that is correct. Run them now to confirm the setup is wired
> correctly and all tests are visible as failures.
>
> The next step is **ddd-implement** — it will implement the function until
> all tests pass."

If the user reports an import error on the implementation file (because it doesn't
exist yet), that is expected. Tell them:

> "The import error on `./check-active` is expected — the file doesn't exist yet.
> You can either create an empty stub now or proceed directly to the
> implementation skill."

Example stub for any function:
```ts
export const checkActive: CheckActiveFn['signature'] = (_input) => ({
    ok: false, errors: ['not_implemented' as any],
})
```

---

## Hard rules

- **The test file contains imports and one `testSpec` call only.** Nothing else,
  except for shell factories which also declare mock deps (minimal inline fakes).
- **No `beforeEach`, `afterEach`, `beforeAll`, `afterAll`.** If setup is needed,
  the spec design is wrong — go back to the spec skill.
- **No custom assertions** in the test file. All assertions live inside `testSpec`
  in `spec-framework.ts`.
- **No inline test data.** All data lives in the spec file.
- **One `testSpec` call per test file.** One file per function.
- **Never modify the spec file** to make the test file simpler.
- **Never add logic** to handle edge cases. If edge cases are missing,
  they belong in the spec file.
- **Import paths must be correct.** Count the relative depth from the test file
  to `shared/spec-framework.ts`. Common mistake: wrong number of `../` segments.
- **If the user asks to add a one-off test** for something not in the spec:
  > "That test belongs in the spec file as a new failure or success example —
  > shall we add it there first? That way it stays part of the validated contract."

## Additional resources

- For project conventions and folder structure, see [../ddd-init/reference.md](../ddd-init/reference.md)
