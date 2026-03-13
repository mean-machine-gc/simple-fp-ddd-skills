# ddd-init — Complete File Contents

Reference implementations for every file created by ddd-init. Each section
corresponds to a step in SKILL.md.

---

## shared/spec-framework.ts (Step 2)

```ts
// shared/spec-framework.ts

// =============================================================================
// Spec Framework — the shared infrastructure
// =============================================================================

// -- Result -------------------------------------------------------------------

export type Result<T, F extends string = string, S extends string = string> =
    | { ok: true; value: T; successType: S[] }
    | { ok: false; errors: F[] }

// -- SpecFn -------------------------------------------------------------------

export type SpecFn<I, O, F extends string, S extends string> = {
    signature: (i: I) => Result<O, F, S>
    asyncSignature: (i: I) => Promise<Result<O, F, S>>
    result: Result<O, F, S>
    input: I
    failures: F
    successTypes: S
    output: O
}
export type AnyFn = SpecFn<any, any, any, any>

// -- StrategyFn ---------------------------------------------------------------
// Phantom type bundle for strategies — enforces that all handlers share
// the same input and output types. The factory dispatch line
// `steps.strategy[tag](input)` is only sound when this holds.
//
// Usage:
//   type DiscountStrategyFn = StrategyFn<DiscountInput, DiscountResult, CouponType, F, S>
//   type Steps = { calculateDiscount: DiscountStrategyFn['handlers'] }

export type StrategyFn<N extends string, I, O, C extends string, F extends string, S extends string> = {
    name: N
    input: I
    output: O
    cases: C
    failures: F
    successTypes: S
    handlers: Record<C, (i: I) => Result<O, F, S>>
}
export type AnyStrategyFn = StrategyFn<any, any, any, any, any, any>

// -- Steps — discriminated union ----------------------------------------------
// Each step type carries only the fields that belong to it.
//   step     → has optional spec (for failure inheritance)
//   dep      → no spec (infrastructure, tested separately)
//   strategy → has handlers (Record of handler specs, keyed by case name)

export type StepStep = {
    name: string
    type: 'step'
    description: string
    spec?: Spec<AnyFn>
}

export type DepStep = {
    name: string
    type: 'dep'
    description: string
}

export type StrategyStep = {
    name: string
    type: 'strategy'
    description: string
    handlers: Record<string, Spec<AnyFn>>
}

export type StepInfo = StepStep | DepStep | StrategyStep

// -- Examples -----------------------------------------------------------------

export type FailExample<Fn extends AnyFn> = {
    description: string
    whenInput: Fn['input']
}

export type SuccessExample<Fn extends AnyFn> = {
    description: string
    whenInput: Fn['input']
    then: Fn['output']
}

// -- Groups -------------------------------------------------------------------

export type FailGroup<Fn extends AnyFn> = {
    description: string
    examples: FailExample<Fn>[]
    coveredBy?: string
}

export type SuccessGroup<Fn extends AnyFn> = {
    description: string
    examples: SuccessExample<Fn>[]
}

// -- Assertions ---------------------------------------------------------------

export type SpecAssert<Fn extends AnyFn> = (input: Fn['input'], output: Fn['output']) => boolean

export type AssertionGroup<Fn extends AnyFn> = {
    [k: string]: {
        description: string
        assert: SpecAssert<Fn>
    }
}

// -- Spec ---------------------------------------------------------------------
// Behavioral contract + optional algorithm decomposition.
// steps is the "how" — visible, reviewable, and the source for auto-inherited failures.
// When steps is present, shouldFailWith is partial — inherited failures are resolved at runtime.
// document: true opts in to .spec.md generation via `npm run gen:specs`.

export type Spec<Fn extends AnyFn> = {
    document?: boolean
    steps?: StepInfo[]
    shouldFailWith: Partial<Record<Fn['failures'], FailGroup<Fn>>>
    shouldSucceedWith: Record<Fn['successTypes'], SuccessGroup<Fn>>
    shouldAssert: Record<Fn['successTypes'], AssertionGroup<Fn>>
}

// -- asStepSpec ---------------------------------------------------------------
// Absorbs the AnyFn erasure cast. Steps with `never` failures or any SpecFn
// variant can be passed to StepInfo.spec without `as unknown as Spec<AnyFn>`.
//
// Before: { name: 'calculateTotal', type: 'step', spec: calculateTotalSpec as unknown as Spec<AnyFn> }
// After:  { name: 'calculateTotal', type: 'step', spec: asStepSpec(calculateTotalSpec) }

export const asStepSpec = <Fn extends AnyFn>(spec: Spec<Fn>): Spec<AnyFn> =>
    spec as unknown as Spec<AnyFn>

// -- inheritFromSteps() -------------------------------------------------------
// Auto-inherits failure groups from all step specs in a steps array.
// Handles three step types:
//   step     → inherits from spec.shouldFailWith, recurses into spec.steps
//   dep      → nothing to inherit
//   strategy → inherits from each handler's spec, keyed by case name

const inheritFromSpec = (
    stepName: string,
    spec: Spec<AnyFn>,
    result: Record<string, FailGroup<any>>,
) => {
    // Direct failures from this spec
    for (const [key, group] of Object.entries(spec.shouldFailWith) as [string, FailGroup<any>][]) {
        if (!(key in result)) {
            result[key] = {
                description: group.description,
                examples: [],
                coveredBy: stepName,
            }
        }
    }
    // Recurse into nested steps (fractal composition)
    if (spec.steps) {
        const nested = inheritFromSteps(spec.steps)
        for (const [key, group] of Object.entries(nested)) {
            if (!(key in result)) {
                result[key] = {
                    ...group,
                    coveredBy: `${stepName} → ${group.coveredBy}`,
                }
            }
        }
    }
}

export const inheritFromSteps = (steps: StepInfo[]): Record<string, FailGroup<any>> => {
    const result: Record<string, FailGroup<any>> = {}

    for (const step of steps) {
        switch (step.type) {
            case 'step':
                if (step.spec) {
                    inheritFromSpec(step.name, step.spec, result)
                }
                break

            case 'strategy':
                for (const [caseName, handlerSpec] of Object.entries(step.handlers)) {
                    inheritFromSpec(`${step.name} (${caseName})`, handlerSpec, result)
                }
                break

            case 'dep':
                // Nothing to inherit — deps are infrastructure
                break
        }
    }

    return result
}

// -- testSpec -----------------------------------------------------------------
// Universal runner. Handles both sync (Fn['signature']) and async (Fn['asyncSignature']).
// Usage:
//   testSpec('checkActive', checkActiveSpec, checkActive)           — sync atomic
//   testSpec('subtractQtyCore', coreSpec, subtractQtyCore)          — sync factory
//   testSpec('subtractQtyShell', shellSpec, subtractQtyShell)       — async factory

export const testSpec = <Fn extends AnyFn>(
    name: string,
    spec: Spec<Fn>,
    fn: Fn['signature'] | Fn['asyncSignature'],
) => {
    // Normalize to async for uniform handling
    const run = async (input: Fn['input']): Promise<Fn['result']> => fn(input) as any

    // -- Resolve failures: merge inherited from steps + explicit overrides -----
    const resolvedFailures: Record<string, FailGroup<Fn>> = {}

    if (spec.steps) {
        const inherited = inheritFromSteps(spec.steps)
        for (const [key, group] of Object.entries(inherited)) {
            resolvedFailures[key] = group
        }
    }

    // Explicit entries override inherited ones
    for (const [key, group] of Object.entries(spec.shouldFailWith) as [string, FailGroup<Fn>][]) {
        if (group) resolvedFailures[key] = group
    }

    describe(name, () => {
        // -- Failures ---------------------------------------------------------
        describe('failures', () => {
            for (const [failure, group] of Object.entries(resolvedFailures) as [Fn['failures'], FailGroup<Fn>][]) {
                if (group.examples.length > 0) {
                    describe(`${failure} — ${group.description}`, () => {
                        for (const example of group.examples) {
                            test(example.description, async () => {
                                const result = await run(example.whenInput)
                                expect(result.ok).toBe(false)
                                if (!result.ok) {
                                    expect(result.errors).toContain(failure)
                                }
                            })
                        }
                    })
                } else if (group.coveredBy) {
                    test.skip(`${failure} — ${group.description} (covered by ${group.coveredBy})`, () => {})
                } else {
                    test.todo(`${failure} — ${group.description}`)
                }
            }
        })

        // -- Successes --------------------------------------------------------
        describe('successes', () => {
            for (const [success, group] of Object.entries(spec.shouldSucceedWith) as [Fn['successTypes'], SuccessGroup<Fn>][]) {
                describe(`${success} — ${group.description}`, () => {
                    for (const example of group.examples) {
                        test(example.description, async () => {
                            const result = await run(example.whenInput)
                            expect(result.ok).toBe(true)
                            if (result.ok) {
                                expect(result.successType).toContain(success)
                                expect(result.value).toEqual(example.then)
                            }
                        })
                    }

                    const assertions = spec.shouldAssert[success]
                    if (assertions) {
                        for (const [_, assertion] of Object.entries(assertions)) {
                            for (const example of group.examples) {
                                test(`${example.description} — ${assertion.description}`, async () => {
                                    const result = await run(example.whenInput)
                                    expect(result.ok).toBe(true)
                                    if (result.ok) {
                                        expect(assertion.assert(example.whenInput, result.value)).toBe(true)
                                    }
                                })
                            }
                        }
                    }
                })
            }
        })
    })
}

// -- CanonicalFn --------------------------------------------------------------
// Standardized implementation structure for flat functions (no decomposition).
// The canonical formula: constraints → conditions → transform.
//
// - constraints: Record<F, predicate>  — returns true when input is VALID
//                                        (constraint satisfied). False → failure.
// - conditions:  Record<S, predicate>  — first match wins. Determines success type.
// - transform:   Record<S, fn>         — produces the output for the matched condition.
//
// Usage:
//   const def: CanonicalFn<CheckActiveFn> = { constraints: {...}, conditions: {...}, transform: {...} }
//   export const checkActive = execCanonical<CheckActiveFn>(def)
//
// The result is a Fn['signature'] — slots directly into factory Steps or testSpec.

export type CanonicalFn<Fn extends AnyFn> = {
    constraints: Record<Fn['failures'], (input: Fn['input']) => boolean>
    conditions:  Record<Fn['successTypes'], (input: Fn['input']) => boolean>
    transform:   Record<Fn['successTypes'], (input: Fn['input']) => Fn['output']>
}

// -- execCanonical ------------------------------------------------------------
// Executes the canonical formula:
//   1. Check all constraints — accumulate failures (predicate returns false → fail)
//   2. Find first matching condition (first match wins — declaration order matters)
//   3. Transform via the matched condition's transform function
//
// Returns Fn['signature'] — a standard (input) => Result<O, F, S> function.

export const execCanonical = <Fn extends AnyFn>(
    def: CanonicalFn<Fn>,
): Fn['signature'] =>
    (input: Fn['input']): Fn['result'] => {
        // 1. Check all constraints — accumulate failures
        const errors: Fn['failures'][] = []
        for (const [failure, predicate] of Object.entries(def.constraints) as [Fn['failures'], (i: Fn['input']) => boolean][]) {
            if (!predicate(input)) errors.push(failure)
        }
        if (errors.length > 0) return { ok: false, errors }

        // 2. Find matching success condition (first match wins)
        for (const [successType, condition] of Object.entries(def.conditions) as [Fn['successTypes'], (i: Fn['input']) => boolean][]) {
            if (condition(input)) {
                // 3. Transform
                const value = def.transform[successType](input)
                return { ok: true, value, successType: [successType] }
            }
        }

        // No condition matched — should not happen if conditions are exhaustive
        return { ok: false, errors: [] as unknown as Fn['failures'][] }
    }
```

---

## scripts/spec-tools.ts (Step 3)

```ts
// scripts/spec-tools.ts

// =============================================================================
// Spec Tools — flatten specs into decision tables
// Strategy-aware: produces main table (linear) + per-handler sub-tables.
// =============================================================================

import type { Spec, StepInfo, StrategyStep } from './spec-framework'

// -- Types --------------------------------------------------------------------

export type FlatConstraint = {
    step: string
    failure: string
    type: 'step' | 'dep' | 'strategy'
}

export type StrategyTable = {
    stepName: string
    caseName: string
    columns: FlatConstraint[]
    successes: string[]
}

export type FlatTable = {
    columns: FlatConstraint[]     // linear constraints only (no strategy handler details)
    successes: string[]
    strategies: StrategyTable[]   // one sub-table per handler
}

// -- flattenSpec --------------------------------------------------------------

export function flattenSpec(spec: any): FlatTable {
    const columns: FlatConstraint[] = []
    const strategies: StrategyTable[] = []

    if (spec.steps) {
        for (const step of spec.steps as StepInfo[]) {
            switch (step.type) {
                case 'step':
                    if (step.spec) {
                        const stepColumns = flattenAtomicSpec(step.name, step.spec)
                        columns.push(...stepColumns)
                    } else {
                        columns.push({ step: step.name, failure: `(${step.name})`, type: 'step' })
                    }
                    break

                case 'dep':
                    columns.push({ step: step.name, failure: `(${step.name})`, type: 'dep' })
                    break

                case 'strategy':
                    // Strategy appears as a single column in the main table
                    columns.push({ step: step.name, failure: `(${step.name})`, type: 'strategy' })

                    // Each handler gets its own sub-table
                    for (const [caseName, handlerSpec] of Object.entries(step.handlers)) {
                        const handlerColumns = flattenAtomicSpec(`${step.name}`, handlerSpec as any)
                        const handlerSuccesses = Object.keys((handlerSpec as any).shouldSucceedWith || {})
                        strategies.push({
                            stepName: step.name,
                            caseName,
                            columns: handlerColumns,
                            successes: handlerSuccesses,
                        })
                    }
                    break
            }
        }
    } else {
        // Atomic spec — collect directly from shouldFailWith
        for (const failure of Object.keys(spec.shouldFailWith || {})) {
            columns.push({ step: '(self)', failure, type: 'step' })
        }
    }

    // Own failures not inherited from steps — insert at the end of the linear columns
    // These are typically dep failures or factory-level validations
    const inheritedKeys = new Set(columns.map(c => c.failure))
    for (const [failure, group] of Object.entries(spec.shouldFailWith || {}) as any[]) {
        if (group && !inheritedKeys.has(failure)) {
            // Use coveredBy if available to attribute the failure to a step
            const stepName = group.coveredBy || '(own)'
            columns.push({ step: stepName, failure, type: 'step' })
        }
    }

    return {
        columns,
        successes: Object.keys(spec.shouldSucceedWith),
        strategies,
    }
}

function flattenAtomicSpec(stepName: string, spec: any): FlatConstraint[] {
    const result: FlatConstraint[] = []

    if (spec.steps) {
        // Composed spec — recurse
        for (const step of spec.steps as StepInfo[]) {
            if (step.type === 'step' && step.spec) {
                const nested = flattenAtomicSpec(step.name, step.spec)
                for (const entry of nested) {
                    result.push({ ...entry, step: `${stepName}.${entry.step}` })
                }
            }
        }
    } else {
        // Atomic — collect from shouldFailWith
        for (const failure of Object.keys(spec.shouldFailWith || {})) {
            result.push({ step: stepName, failure, type: 'step' })
        }
    }

    return result
}

// -- toMainTable --------------------------------------------------------------
// Linear pipeline table. Strategy steps appear as a single column (pass/fail).
// Strategy handler constraints are NOT shown here — they're in sub-tables.

export function toMainTable(table: FlatTable): string {
    const { columns, successes, strategies } = table

    const realColumns = columns.filter(c => !c.failure.startsWith('(') || c.type === 'strategy')

    const header = [
        'Scenario',
        ...realColumns.map(c =>
            c.type === 'strategy'
                ? `\`${c.step}\` _(strategy)_`
                : `\`${c.step}\` :${c.failure}`
        ),
        'Outcome',
    ]
    const separator = ['---', ...realColumns.map(() => ':---:'), '---']

    const successRows = successes.map(s => [
        `OK ${s}`,
        ...realColumns.map(c =>
            c.type === 'strategy'
                ? `pass _(${findHandlerForSuccess(strategies, c.step, s)})_`
                : 'pass'
        ),
        s,
    ])

    const failureRows = realColumns
        .filter(c => c.type !== 'strategy')
        .map((c, _i) => {
            const colIndex = realColumns.indexOf(c)
            return [
                `FAIL ${c.failure}`,
                ...realColumns.map((_, j) =>
                    j < colIndex ? 'pass' : j === colIndex ? 'FAIL' : '--'
                ),
                `Fails: \`${c.failure}\``,
            ]
        })

    // Strategy note rows
    const strategyNoteRows = strategies.length > 0
        ? [[
            `_strategy_`,
            ...realColumns.map(() => ''),
            `_See handler tables below_`,
        ]]
        : []

    const rows = [header, separator, ...successRows, ...failureRows, ...strategyNoteRows]
    return rows.map(r => `| ${r.join(' | ')} |`).join('\n')
}

function findHandlerForSuccess(strategies: StrategyTable[], stepName: string, success: string): string {
    for (const s of strategies) {
        if (s.stepName === stepName && s.successes.includes(success)) {
            return s.caseName
        }
    }
    return '?'
}

// -- toHandlerTables ----------------------------------------------------------
// One table per strategy handler. Shows the handler's own constraints.

export function toHandlerTables(table: FlatTable): string {
    if (table.strategies.length === 0) return ''

    const sections: string[] = []

    // Group by strategy step name
    const byStep = new Map<string, StrategyTable[]>()
    for (const s of table.strategies) {
        const list = byStep.get(s.stepName) || []
        list.push(s)
        byStep.set(s.stepName, list)
    }

    for (const [stepName, handlers] of byStep) {
        sections.push(`### Strategy: \`${stepName}\`\n`)

        for (const handler of handlers) {
            sections.push(`#### Handler: \`${handler.caseName}\`\n`)

            if (handler.columns.length === 0) {
                sections.push('_No constraints — handler always succeeds._\n')
                continue
            }

            const header = [
                'Scenario',
                ...handler.columns.map(c => `\`${c.step}\` :${c.failure}`),
                'Outcome',
            ]
            const separator = ['---', ...handler.columns.map(() => ':---:'), '---']

            const successRows = handler.successes.map(s => [
                `OK ${s}`,
                ...handler.columns.map(() => 'pass'),
                s,
            ])

            const failureRows = handler.columns.map((c, i) => [
                `FAIL ${c.failure}`,
                ...handler.columns.map((_, j) => j < i ? 'pass' : j === i ? 'FAIL' : '--'),
                `Fails: \`${c.failure}\``,
            ])

            const rows = [header, separator, ...successRows, ...failureRows]
            sections.push(rows.map(r => `| ${r.join(' | ')} |`).join('\n'))
            sections.push('')
        }
    }

    return sections.join('\n')
}

// -- toStepTable --------------------------------------------------------------

export function toStepTable(spec: any): string {
    if (!spec.steps) return '_Atomic function — no pipeline steps._'

    const rows: string[][] = [
        ['#', 'Name', 'Type', 'Description', 'Failure Codes'],
        ['---', '---', '---', '---', '---'],
    ]

    for (let i = 0; i < spec.steps.length; i++) {
        const step = spec.steps[i]
        let failures: string[] = []
        let typeStr: string

        switch (step.type) {
            case 'step':
                typeStr = '`STEP`'
                if (step.spec) {
                    failures = Object.keys(step.spec.shouldFailWith || {})
                }
                break
            case 'dep':
                typeStr = '`DEP`'
                break
            case 'strategy':
                typeStr = '`STRATEGY`'
                // Collect failures from all handlers
                if (step.handlers) {
                    for (const [caseName, handlerSpec] of Object.entries(step.handlers) as any[]) {
                        const handlerFailures = Object.keys(handlerSpec.shouldFailWith || {})
                        failures.push(...handlerFailures.map(f => `${f} _(${caseName})_`))
                    }
                }
                break
        }

        const failStr = failures.length > 0
            ? failures.map(f => f.startsWith('`') ? f : `\`${f}\``).join(', ')
            : '--'

        rows.push([String(i + 1), `\`${step.name}\``, typeStr!, step.description, failStr])
    }

    return rows.map(r => `| ${r.join(' | ')} |`).join('\n')
}

// -- buildSpecMd --------------------------------------------------------------
// Assembles the full .spec.md with strategy-aware tables.

export function buildSpecMd(name: string, spec: any): string {
    const pipeline = toStepTable(spec)
    const table = flattenSpec(spec)
    const main = toMainTable(table)
    const handlers = toHandlerTables(table)

    const parts = [
        `# ${name}`,
        '',
        `> Auto-generated from \`${name}.spec.ts\`. Do not edit — run \`npm run gen:specs\` to regenerate.`,
        `> For business-friendly documentation, see \`/docs/\`.`,
        '',
        '---',
        '',
        '## Pipeline',
        '',
        pipeline,
        '',
        '---',
        '',
        '## Decision Table',
        '',
        main,
    ]

    if (handlers) {
        parts.push('', '---', '', handlers)
    }

    return parts.join('\n') + '\n'
}
```

---

## scripts/generate-specs.ts (Step 4)

Auto-discovers specs with `document: true` via glob — no manual manifest needed.
Fully generated structural docs — pipeline tables and decision tables.
Overwrites the `.spec.md` on every run. No prose, no markers, no manual editing.

```ts
// scripts/generate-specs.ts

import { writeFileSync } from 'fs'
import { globSync } from 'glob'
import { resolve, basename } from 'path'
import { buildSpecMd } from './spec-tools'

async function main() {
  const specFiles = globSync('src/**/*.spec.ts')

  if (specFiles.length === 0) {
    console.log('No .spec.ts files found.')
    return
  }

  let generated = 0

  for (const file of specFiles) {
    const resolvedPath = resolve(file)
    const mod = await import(resolvedPath)

    // Find exported specs with document: true
    for (const [exportName, value] of Object.entries(mod)) {
      if (
        value &&
        typeof value === 'object' &&
        'document' in value &&
        (value as any).document === true
      ) {
        const name = basename(file, '.spec.ts')
        const mdPath = resolvedPath.replace(/\.spec\.ts$/, '.spec.md')
        const content = buildSpecMd(name, value)
        writeFileSync(mdPath, content)
        console.log(`  ${name} (${exportName}): wrote ${mdPath}`)
        generated++
      }
    }
  }

  if (generated === 0) {
    console.log('No specs with document: true found — nothing to generate.')
  } else {
    console.log(`\nGenerated ${generated} .spec.md file(s).`)
  }
}

main().catch(err => {
  console.error('generate-specs failed:', err)
  process.exit(1)
})
```

---

## scripts/tsconfig.json (Step 6)

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "esModuleInterop": true,
    "strict": true,
    "skipLibCheck": true,
    "outDir": "./dist",
    "rootDir": "."
  },
  "include": ["./**/*.ts"]
}
```

---

## docs/ directory (Step 7)

### docs/_config.yml

```yaml
title: Domain Specs
description: Business-friendly operation specifications
theme: just-the-docs

mermaid:
  version: "11"

nav_external_links: []
```

### docs/index.md

The domain home page is a living document — updated by the `ddd-documentation`
skill each time a new aggregate or operation is added. The init skill creates
a minimal placeholder:

```md
---
layout: default
title: Home
nav_order: 1
mermaid: true
---

# Domain Specifications

> Brief description of what this domain layer does.

---

*This page is populated by the ddd-documentation skill as operations are documented.*
```
