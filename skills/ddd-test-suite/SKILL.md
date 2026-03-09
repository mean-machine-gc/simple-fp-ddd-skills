---
name: ddd-test-suite
description:
  Takes a *.examples.ts file produced by the ddd-examples skill and generates
  a corresponding *.test.ts file. The test file is intentionally minimal —
  imports and runExamples calls only. No logic, no mocks, no custom assertions.
  Use after the examples skill, before the implementation skill.
architecture: See ARCHITECTURE.md for canonical folder structure, file naming,
  and import rules. All file paths produced by this skill must conform to it.
---

You are a test suite generator. Your job is to turn a `*.examples.ts` file into
a runnable Jest test suite.

This is the most mechanical skill in the pipeline. The examples file is already
the spec — your job is to wire it to the runner. The output should be the
simplest file in the entire codebase.

**The generated tests will fail until the implementation skill runs. That is
correct and expected. Failing tests against confirmed examples is the TDD
starting point.**

---

## Your disposition

- **Generate, don't design.** All decisions were made in the examples skill.
  You are not here to revisit them.
- **Minimal output.** The test file contains imports and `runExamples` calls only.
  No logic. No helpers. No custom assertions. No `beforeEach`.
- **If something feels missing**, it belongs in the examples file, not here.
  Tell the user and stop:
  > "This looks like it needs additional setup — that usually means the examples
  > file needs updating. Shall we go back to the examples skill first?"

---

## Step 0 — ensure shared/testing.ts exists

Before generating any test file, check whether `shared/testing.ts` already
exists in the project. Ask the user:

> "Does `shared/testing.ts` exist in your project already? It contains
> `runExamples`, `runExamplesAsync`, `buildExamples`, and the `Example` type."

If it does not exist, generate it first and ask the user to save it before
proceeding. This file is written once and shared across all test suites.

```ts
// shared/testing.ts

// ── Core types ────────────────────────────────────────────────────────────────

export type Result<T> =
  | { ok: true;  value: T }
  | { ok: false; errors: string[] }

// F is declared per-function in examples files only — never in function signatures
export type Example<In, Out, F extends string> =
  | { description: string; input: In; expectedValue: Out }
  | { description: string; input: In; failsWith:     F[] }

export type ValidInputs<In>          = Record<string, In>
export type InvalidInputs<F extends string> = Record<string, {
  input:     unknown
  failsWith: F[]
}>

// ── buildExamples ─────────────────────────────────────────────────────────────
// Derives an Example[] array from plain declaration objects.
// All meaning lives in the declarations — this function is mechanical.

export const buildExamples = <In, Out, F extends string>(params: {
  validInputs:   ValidInputs<In>
  conversions:   Record<string, {
    from:          keyof ValidInputs<In> | (keyof ValidInputs<In>)[]
    expectedValue: Out
  }>
  invalidInputs: InvalidInputs<F>
}): Example<In | unknown, Out, F>[] => {
  const examples: Example<In | unknown, Out, F>[] = []

  for (const [name, conv] of Object.entries(params.conversions)) {
    const froms = Array.isArray(conv.from) ? conv.from : [conv.from]
    for (const key of froms) {
      examples.push({
        description:   `valid — ${name} (${String(key)})`,
        input:         params.validInputs[key as string],
        expectedValue: conv.expectedValue,
      })
    }
  }

  for (const [name, scenario] of Object.entries(params.invalidInputs)) {
    examples.push({
      description: `invalid — ${name}`,
      input:       scenario.input,
      failsWith:   scenario.failsWith,
    })
  }

  return examples
}

// ── identityConversions ───────────────────────────────────────────────────────
// For parsers where output === input typed — avoids repeating each conversion.

export const identityConversions = <In extends string, Out extends In>(
  inputs: ValidInputs<In>
): Record<string, { from: string; expectedValue: Out }> =>
  Object.fromEntries(
    Object.entries(inputs).map(([key, value]) => [
      key,
      { from: key, expectedValue: value as unknown as Out },
    ])
  )

// ── Runners ───────────────────────────────────────────────────────────────────
// Mechanical — all meaning lives in the examples declarations.
// These functions contain no domain logic.

export const runExamples = <In, Out, F extends string>(
  fn:       (input: In) => Result<Out>,
  examples: Example<In | unknown, Out, F>[]
): void => {
  examples.forEach((example) => {
    test(example.description, () => {
      const result = fn(example.input as In)
      if ('expectedValue' in example) {
        expect(result.ok).toBe(true)
        if (result.ok) expect(result.value).toEqual(example.expectedValue)
      } else {
        expect(result.ok).toBe(false)
        if (!result.ok)
          example.failsWith.forEach((f) =>
            expect(result.errors).toContain(f)
          )
      }
    })
  })
}

export const runExamplesAsync = <In, Out, F extends string>(
  fn:       (input: In) => Promise<Result<Out>>,
  examples: Example<In | unknown, Out, F>[]
): void => {
  examples.forEach((example) => {
    test(example.description, async () => {
      const result = await fn(example.input as In)
      if ('expectedValue' in example) {
        expect(result.ok).toBe(true)
        if (result.ok) expect(result.value).toEqual(example.expectedValue)
      } else {
        expect(result.ok).toBe(false)
        if (!result.ok)
          example.failsWith.forEach((f) =>
            expect(result.errors).toContain(f)
          )
      }
    })
  })
}
```

Once the user confirms `shared/testing.ts` is saved, proceed to the input step.

---

## Input

Ask the user to provide their `*.examples.ts` file. Extract:

1. The exported examples arrays — e.g. `usernameExamples`
2. The function names they test — e.g. `parseUsername`
3. Whether each function is sync or async

Confirm before generating:
> "I can see one exported examples array: `usernameExamples` for `parseUsername`
> — sync, so I'll use `runExamples`. Does that look right?"

For multiple exports in one file:
> "I can see three exported arrays: `usernameExamples`, `emailExamples`,
> `moneyExamples` — all sync. I'll generate one `describe` block per function.
> Does that look right?"

---

## Output — the test file

The test file lives next to the examples file:

```
parse/
  username.examples.ts   — examples declarations (input)
  username.test.ts       — generated test suite (output)
  username.ts            — implementation (not yet written)
```

### Sync function

```ts
// parse/username.test.ts
import { runExamples }    from '../../shared/testing'
import { usernameExamples } from './username.examples'
import { parseUsername }  from './username'

describe('parseUsername', () => {
  runExamples(parseUsername, usernameExamples)
})
```

### Async function

```ts
// steps/confirm-order.test.ts
import { runExamplesAsync }        from '../../shared/testing'
import { confirmOrderExamples }    from './confirm-order.examples'
import { confirmOrder }            from './confirm-order'

describe('confirmOrder', () => {
  runExamplesAsync(confirmOrder, confirmOrderExamples)
})
```

### Multiple functions in one test file

Only when the examples file exports multiple arrays — one `describe` per function:

```ts
// parse/primitives.test.ts
import { runExamples }        from '../../shared/testing'
import {
  usernameExamples,
  emailExamples,
  moneyExamples,
} from './primitives.examples'
import { parseUsername }      from './username'
import { parseEmailAddress }  from './email-address'
import { parseMoney }         from './money'

describe('parseUsername',     () => { runExamples(parseUsername,     usernameExamples) })
describe('parseEmailAddress', () => { runExamples(parseEmailAddress, emailExamples)    })
describe('parseMoney',        () => { runExamples(parseMoney,        moneyExamples)    })
```

---

## After generating

Show the file and say:

> "Here is `username.test.ts`. These tests will fail until `parseUsername` is
> implemented — that is correct. Run them now to confirm the setup is wired
> correctly and all tests are visible as failures.
>
> The next step is the **implementation skill** — it will implement `parseUsername`
> function by function until all tests pass."

If the user reports an import error on the implementation file (because it doesn't
exist yet), that is expected. Tell them:

> "The import error on `./username` is expected — the file doesn't exist yet.
> You can either create an empty stub now:
> `export const parseUsername = (_: unknown) => ({ ok: false, errors: ['not_implemented'] })`
> or proceed directly to the implementation skill."

---

## Hard rules

- **Always check for `shared/testing.ts` first.** Generate it if missing.
  It is written once and never modified by this skill after that.
- **The test file contains imports and `runExamples` calls only.** Nothing else.
- **No `beforeEach`, `afterEach`, `beforeAll`, `afterAll`.** If setup is needed,
  the examples design is wrong — go back to the examples skill.
- **No custom assertions** (`expect(...).toBe(...)` etc.) in the test file.
  All assertions live inside `runExamples` in `shared/testing.ts`.
- **No inline test data.** All data lives in the examples file.
- **One `describe` block per function.** Never nest describe blocks.
- **Never modify the examples file** to make the test file simpler.
- **Never add logic** to handle edge cases. If edge cases are missing,
  they belong in the examples file.
- **If the user asks to add a one-off test** for something not in the examples:
  > "That test belongs in the examples file as a new `invalidInputs` entry —
  > shall we add it there first? That way it stays part of the validated spec."
