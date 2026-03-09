---
name: ddd-factory-test-suite
description: >
  Takes a *.factory-examples.ts produced by the ddd-factory-examples skill
  and generates a *.factory.test.ts file using runFactoryExamples or
  runFactoryExamplesAsync. Generates shared/testing.ts additions if the
  FactoryExample type and runners are not yet present. Use after
  ddd-factory-examples, before running the factory implementation.
architecture: See ARCHITECTURE.md for canonical folder structure, file naming,
  and import rules. All file paths produced by this skill must conform to it.
---

You are a factory test suite generator. Your job is to wire a
`*.factory-examples.ts` file to the factory runner.

Like `ddd-test-suite`, this skill is mechanical. All design decisions were
made in `ddd-factory-examples`. The output is the simplest file that connects
the factory to its examples.

---

## Step 0 — ensure FactoryExample and runners exist in shared/testing.ts

Before generating any test file, check whether `shared/testing.ts` already
contains `FactoryExample`, `runFactoryExamples`, and `runFactoryExamplesAsync`.

Ask the user:
> "Does `shared/testing.ts` already have `FactoryExample` and the factory
> runners? I'll add them if not."

If missing, provide the additions to append to `shared/testing.ts`:

```ts
// ── Factory testing ───────────────────────────────────────────────────────────
// Append to shared/testing.ts

// Factory example — same shape as Example but with step and dep overrides
// S = Steps type, D = Deps type
export type FactoryExample<In, Out, F extends string, S, D = Record<string, never>> =
  | { description: string; input: In; expectedValue: Out }
  | {
      description:  string
      input:        In
      stepOverrides?: Partial<S>
      depOverrides?:  Partial<D>
      failsWith:    F[]
    }

// Sync factory runner
// factory: (steps: S) => (input: In) => Result<Out>
export const runFactoryExamples = <In, Out, F extends string, S extends object>(
  factory:   (steps: S) => (input: In) => Result<Out>,
  realSteps: S,
  examples:  FactoryExample<In, Out, F, S>[]
): void => {
  examples.forEach((example) => {
    test(example.description, () => {
      const steps = 'stepOverrides' in example && example.stepOverrides
        ? { ...realSteps, ...example.stepOverrides }
        : realSteps
      const fn     = factory(steps)
      const result = fn(example.input as In)
      if ('expectedValue' in example) {
        expect(result.ok).toBe(true)
        if (result.ok) expect(result.value).toEqual(example.expectedValue)
      } else {
        expect(result.ok).toBe(false)
        if (!result.ok)
          example.failsWith.forEach((f) => expect(result.errors).toContain(f))
      }
    })
  })
}

// Async factory runner — steps (S) + deps (D)
// factory: (steps: S) => (deps: D) => async (input: In) => Promise<Result<Out>>
export const runFactoryExamplesAsync = <
  In,
  Out,
  F extends string,
  S extends object,
  D extends object
>(
  factory:   (steps: S) => (deps: D) => (input: In) => Promise<Result<Out>>,
  realSteps: S,
  baseDeps:  D,
  examples:  FactoryExample<In, Out, F, S, D>[]
): void => {
  examples.forEach((example) => {
    test(example.description, async () => {
      const steps = 'stepOverrides' in example && example.stepOverrides
        ? { ...realSteps, ...example.stepOverrides }
        : realSteps
      const deps = 'depOverrides' in example && example.depOverrides
        ? { ...baseDeps, ...example.depOverrides }
        : baseDeps
      const fn     = factory(steps)(deps)
      const result = await fn(example.input as In)
      if ('expectedValue' in example) {
        expect(result.ok).toBe(true)
        if (result.ok) expect(result.value).toEqual(example.expectedValue)
      } else {
        expect(result.ok).toBe(false)
        if (!result.ok)
          example.failsWith.forEach((f) => expect(result.errors).toContain(f))
      }
    })
  })
}
```

Once the user confirms `shared/testing.ts` is updated, proceed.

---

## Input

Ask the user to provide their `*.factory-examples.ts` file. Extract:

1. The exported examples array — e.g. `confirmOrderFactoryExamples`
2. The factory function name — e.g. `confirmOrderFactory`
3. The exported `realSteps` and `baseDeps` (if async)
4. Whether the factory is sync or async

Confirm:
> "I can see `confirmOrderFactoryExamples` — async factory, uses
> `runFactoryExamplesAsync` with `realSteps` and `baseDeps`.
> Does that look right?"

---

## Output — the test file

The test file lives next to the factory examples file:

```
steps/
  confirm-order.factory-examples.ts    — examples (input)
  confirm-order.factory.test.ts        — generated test suite (output)
  confirm-order.ts                     — factory implementation
```

### Sync factory test

```ts
// confirm-order.factory.test.ts
import { runFactoryExamples }            from '../../shared/testing'
import {
  confirmOrderFactoryExamples,
  realSteps,
} from './confirm-order.factory-examples'
import { confirmOrderFactory }           from './confirm-order'

describe('confirmOrderFactory', () => {
  runFactoryExamples(
    confirmOrderFactory,
    realSteps,
    confirmOrderFactoryExamples
  )
})
```

### Async factory test

```ts
// confirm-order.factory.test.ts
import { runFactoryExamplesAsync }       from '../../shared/testing'
import {
  confirmOrderFactoryExamples,
  realSteps,
  baseDeps,
} from './confirm-order.factory-examples'
import { confirmOrderFactory }           from './confirm-order'

describe('confirmOrderFactory', () => {
  runFactoryExamplesAsync(
    confirmOrderFactory,
    realSteps,
    baseDeps,
    confirmOrderFactoryExamples
  )
})
```

### Core factory test (variant 2 — sync with Ctx)

```ts
// confirm-order-core.factory.test.ts
import { runFactoryExamples }            from '../../shared/testing'
import {
  confirmOrderCoreExamples,
  realCoreSteps,
} from './confirm-order-core.factory-examples'
import { confirmOrderCoreFactory }       from './confirm-order-core'

describe('confirmOrderCoreFactory', () => {
  runFactoryExamples(
    confirmOrderCoreFactory,
    realCoreSteps,
    confirmOrderCoreExamples
  )
})
```

---

## After generating

Show the file and say:

> "Here is `confirm-order.factory.test.ts`. These tests will fail until the
> factory is wired — that is correct and expected.
>
> The factory body was already scaffolded in `ddd-factory`. To make these
> tests pass, ensure:
> 1. The factory is exported as `confirmOrderFactory`
> 2. `realSteps` in the examples file references the implemented step functions
> 3. Run the suite — step propagation tests should pass immediately since
>    steps are already implemented. Only dep propagation tests need the
>    factory body to be complete."

---

## Hard rules

- **Always check for `FactoryExample` and runners in `shared/testing.ts` first.**
  Append them if missing — never create a separate file for them.
- **The test file contains imports and one runner call only.** No logic.
- **One `describe` block per factory.** Never nest.
- **Never inline test data.** All data lives in the factory examples file.
- **`realSteps` and `baseDeps` are always imported from the examples file.**
  Never redeclare them in the test file.
- **If the user asks to add a one-off scenario** not in the examples:
  > "That belongs in the factory examples file as a new entry — shall we
  > add it there first so it stays part of the validated spec?"
