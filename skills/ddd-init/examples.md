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

export type Spec<Fn extends AnyFn> = {
    steps?: StepInfo[]
    shouldFailWith: Partial<Record<Fn['failures'], FailGroup<Fn>>>
    shouldSucceedWith: Record<Fn['successTypes'], SuccessGroup<Fn>>
    shouldAssert: Record<Fn['successTypes'], AssertionGroup<Fn>>
}

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

## scripts/spec-manifest.ts (Step 4)

```ts
// scripts/spec-manifest.ts
//
// Add one entry per composed spec (factory). The generate-specs script reads
// this manifest, imports each spec, and writes the .spec.md decision tables
// next to the spec file.
//
// Output path is derived from specPath: same directory, .spec.md extension.

export type ManifestEntry = {
  name:       string   // human-readable name for logging
  specPath:   string   // import path relative to this file (no extension)
  exportName: string   // named export from the spec module
}

export const specManifest: ManifestEntry[] = [
  // Example:
  // {
  //   name: 'subtract-quantity-shell',
  //   specPath: '../src/cart/subtract-quantity/subtract-quantity.spec',
  //   exportName: 'subtractQuantityShellSpec',
  // },
]
```

---

## scripts/generate-specs.ts (Step 5)

Fully generated structural docs — pipeline tables and decision tables.
Overwrites the `.spec.md` on every run. No prose, no markers, no manual editing.

```ts
// scripts/generate-specs.ts

import { writeFileSync } from 'fs'
import { buildSpecMd } from './spec-tools'
import { specManifest } from './spec-manifest'

// -- Manifest validation ------------------------------------------------------

function validateManifest(): { valid: typeof specManifest; errors: string[] } {
  const valid: typeof specManifest = []
  const errors: string[] = []

  for (const entry of specManifest) {
    let resolvedPath: string
    try {
      resolvedPath = require.resolve(entry.specPath)
    } catch {
      errors.push(`x ${entry.name}: file not found — ${entry.specPath}`)
      continue
    }

    // eslint-disable-next-line @typescript-eslint/no-var-requires
    const mod = require(resolvedPath)
    if (!mod[entry.exportName]) {
      errors.push(`x ${entry.name}: export '${entry.exportName}' not found in ${resolvedPath}`)
      continue
    }

    valid.push(entry)
  }

  return { valid, errors }
}

async function main() {
  if (specManifest.length === 0) {
    console.log('spec-manifest.ts is empty — no specs to generate.')
    return
  }

  const { valid: validEntries, errors: manifestErrors } = validateManifest()
  if (manifestErrors.length > 0) {
    console.error('\nManifest validation failed:')
    manifestErrors.forEach(e => console.error(`  ${e}`))
    console.error(`\nFix the manifest (scripts/spec-manifest.ts) and re-run.\n`)
    process.exit(1)
  }

  for (const entry of validEntries) {
    const mod = await import(entry.specPath)
    const spec = mod[entry.exportName]

    const specFile = require.resolve(entry.specPath)
    const mdPath = specFile.replace(/\.spec\.ts$/, '.spec.md')

    const content = buildSpecMd(entry.name, spec)
    writeFileSync(mdPath, content)
    console.log(`  ${entry.name}: wrote ${mdPath}`)
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

nav_external_links: []
```

### docs/index.md

```md
---
title: Home
nav_order: 1
---

# Domain Specifications

Business-friendly documentation for all domain operations.

Each aggregate has its own section. Each operation within an aggregate
has a dedicated page covering overview, interface, business scenarios,
pipeline, and decision tables.

---

*Generated with [ddd-documentation](../.claude/skills-v4/ddd-documentation/SKILL.md).*
```
