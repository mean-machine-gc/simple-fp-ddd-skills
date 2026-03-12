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

// -- Steps --------------------------------------------------------------------

export type StepInfo = {
    name: string
    type: 'step' | 'dep' | 'strategy'
    description: string
    spec?: Spec<AnyFn>
}

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
// Each step's failures get empty examples + coveredBy set to step name.
// Recurses into nested step specs (fractal composition).

export const inheritFromSteps = (steps: StepInfo[]): Record<string, FailGroup<any>> => {
    const result: Record<string, FailGroup<any>> = {}
    for (const step of steps) {
        if (!step.spec) continue
        // Direct failures from this step
        for (const [key, group] of Object.entries(step.spec.shouldFailWith) as [string, FailGroup<any>][]) {
            if (!(key in result)) {
                result[key] = {
                    description: group.description,
                    examples: [],
                    coveredBy: step.name,
                }
            }
        }
        // Recurse into nested steps
        if (step.spec.steps) {
            const nested = inheritFromSteps(step.spec.steps)
            for (const [key, group] of Object.entries(nested)) {
                if (!(key in result)) {
                    result[key] = {
                        ...group,
                        coveredBy: `${step.name} -> ${group.coveredBy}`,
                    }
                }
            }
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

// -- Types --------------------------------------------------------------------

export type FlatConstraint = {
  step:    string    // 'parseCartId', 'checkActive', 'findCart'
  failure: string    // 'not_a_string', 'cart_empty', 'cart_not_found'
  type:    'step' | 'dep' | 'strategy'
}

export type FlatTable = {
  columns:   FlatConstraint[]
  successes: string[]
}

// -- flattenSpec --------------------------------------------------------------
// Recursively walks a Spec's steps array, collecting failures in pipeline order.

export function flattenSpec(spec: any): FlatTable {
  return {
    columns:   flattenSteps(spec),
    successes: Object.keys(spec.shouldSucceedWith),
  }
}

function flattenSteps(spec: any): FlatConstraint[] {
  const result: FlatConstraint[] = []

  if (!spec.steps) {
    // Atomic spec — collect directly from shouldFailWith
    for (const failure of Object.keys(spec.shouldFailWith || {})) {
      result.push({ step: '(self)', failure, type: 'step' })
    }
    return result
  }

  for (const step of spec.steps as any[]) {
    if (step.spec) {
      if (step.spec.steps) {
        // Recursive — this step is itself a composed spec (e.g. core inside shell)
        const nested = flattenSteps(step.spec)
        for (const entry of nested) {
          result.push({ ...entry, step: `${step.name}.${entry.step}` })
        }
      } else {
        // Atomic step spec — collect from shouldFailWith
        for (const failure of Object.keys(step.spec.shouldFailWith || {})) {
          result.push({ step: step.name, failure, type: step.type })
        }
      }
    } else {
      // Dep or step without spec — check if parent spec declares failures for it
      // These are own failures declared directly in the composed spec's shouldFailWith
      result.push({ step: step.name, failure: `(${step.name})`, type: step.type })
    }
  }

  // Also collect own failures from the composed spec that aren't inherited
  // (e.g. cart_not_found, product_not_in_cart)
  const inheritedKeys = new Set(result.map(r => r.failure))
  for (const [failure, group] of Object.entries(spec.shouldFailWith || {}) as any[]) {
    if (group && !inheritedKeys.has(failure)) {
      // Determine which step owns this failure based on coveredBy, or mark as own
      result.push({ step: '(own)', failure, type: 'step' })
    }
  }

  return result
}

// -- toMarkdownTable ----------------------------------------------------------
// Converts flat table to decision table markdown with symbols.

export function toMarkdownTable(table: FlatTable): string {
  const { columns, successes } = table

  // Filter out placeholder entries
  const realColumns = columns.filter(c => !c.failure.startsWith('('))

  const header = [
    'Scenario',
    ...realColumns.map(c => `\`${c.step}\` :${c.failure}`),
    'Outcome',
  ]
  const separator = ['---', ...realColumns.map(() => ':---:'), '---']

  const successRows = successes.map(s => [
    `OK ${s}`,
    ...realColumns.map(() => 'pass'),
    s,
  ])

  const failureRows = realColumns.map((c, i) => [
    `FAIL ${c.failure}`,
    ...realColumns.map((_, j) => j < i ? 'pass' : j === i ? 'FAIL' : '--'),
    `Fails: \`${c.failure}\``,
  ])

  const rows = [header, separator, ...successRows, ...failureRows]
  return rows.map(r => `| ${r.join(' | ')} |`).join('\n')
}

// -- toStepTable --------------------------------------------------------------
// Extracts pipeline step table from a spec's steps array.

export function toStepTable(spec: any): string {
  if (!spec.steps) return '_Atomic function — no pipeline steps._'

  const rows: string[][] = [
    ['#', 'Name', 'Type', 'Description', 'Failure Codes'],
    ['---', '---', '---', '---', '---'],
  ]

  for (let i = 0; i < spec.steps.length; i++) {
    const step = spec.steps[i]
    let failures: string[] = []

    if (step.spec) {
      if (step.spec.steps) {
        failures = flattenSteps(step.spec)
          .filter(c => !c.failure.startsWith('('))
          .map(c => c.failure)
      } else {
        failures = Object.keys(step.spec.shouldFailWith || {})
      }
    }

    const failStr = failures.length > 0
      ? failures.map(f => `\`${f}\``).join(', ')
      : '--'
    const typeStr = `\`${step.type.toUpperCase()}\``

    rows.push([String(i + 1), `\`${step.name}\``, typeStr, step.description, failStr])
  }

  return rows.map(r => `| ${r.join(' | ')} |`).join('\n')
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

import { writeFileSync, mkdirSync } from 'fs'
import { resolve } from 'path'
import { flattenSpec, toMarkdownTable, toStepTable } from './spec-tools'
import { specManifest } from './spec-manifest'

function buildSpecMd(name: string, spec: any): string {
  const pipeline = toStepTable(spec)
  const table = flattenSpec(spec)
  const decision = toMarkdownTable(table)

  return `# ${name}

> Auto-generated from \`${name}.spec.ts\`. Do not edit — run \`npm run gen:specs\` to regenerate.
> For business-friendly documentation, see \`/docs/\`.

---

## Pipeline

${pipeline}

---

## Decision Table

${decision}
`
}

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
