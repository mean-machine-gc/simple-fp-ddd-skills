# ddd-init — Complete File Contents

Reference implementations for every file created by ddd-init. Each section
corresponds to a step in SKILL.md.

---

## shared/spec.ts (Step 2)

```ts
// shared/spec.ts

// ── Result ──────────────────────────────────────────────────────────────────

export type Result<T, F extends string = string, S extends string = string> =
  | { ok: true; value: T; successType: S[] }
  | { ok: false; errors: F[] }

// ── Spec Predicates ─────────────────────────────────────────────────────────

export type ConstraintPredicate<In> =
  (args: { input: In }) => boolean

export type ConditionPredicate<In, Out> =
  (args: { input: In; output: Out }) => boolean

export type AssertionPredicate<In, Out> =
  (args: { input: In; output: Out }) => boolean

// ── Example Types ───────────────────────────────────────────────────────────

export type FailureExample<In> = { when: In }
export type SuccessExample<In, Out> = { when: In; then: Out }
export type MixedFailureExample<In, F extends string> = {
  description: string
  when: In
  failsWith: F[]
}
export type DepPropagationExample<In> = { when: In; failsWith: string }

// Non-empty tuple — compiler guarantees at least one example per key
export type OneOrMore<T> = [T, ...T[]]

// ── FunctionSpec — atomic functions ─────────────────────────────────────────
// Each constraint has its predicate AND its failure examples together.
// Each success type has its condition, assertions, AND its success examples.
// The rule and its proof live in one object.

export type FunctionSpec<In, Out, F extends string, S extends string> = {
  constraints: Record<F, {
    predicate: ConstraintPredicate<In>
    examples: OneOrMore<FailureExample<In>>
  }>
  successes: Record<S, {
    condition: ConditionPredicate<In, Out>
    assertions: Record<string, AssertionPredicate<In, Out>>
    examples: OneOrMore<SuccessExample<In, Out>>
  }>
  mixed?: MixedFailureExample<In, F>[]
}

// ── FactorySpec — factories that compose steps ──────────────────────────────
// No constraint predicates (inherited from step specs). Failure examples are
// factory-level inputs that trigger step failures through real steps.
// No assertions — expected value match suffices for factories.
// Steps can be atomic (FunctionSpec) or nested factories (FactorySpec).

export type StepSpec =
  | FunctionSpec<any, any, string, string>
  | FactorySpec<any, any, string, string>

export type FactorySpec<In, Out, F extends string, S extends string> = {
  steps: Record<string, StepSpec>
  deps?: Record<string, { failures: string[] }>
  failures: Record<F, OneOrMore<FailureExample<In>>>
  successes: Record<S, {
    condition: ConditionPredicate<In, Out>
    examples: OneOrMore<SuccessExample<In, Out>>
  }>
  mixed?: MixedFailureExample<In, F>[]
}

// ── ShellFactorySpec — shell factories with dep propagation ─────────────────

export type ShellFactorySpec<
  In, Out, F extends string, S extends string,
  Deps extends Record<string, any>
> = FactorySpec<In, Out, F, S> & {
  baseDeps: Deps
  depPropagation: Record<keyof Deps, DepPropagationExample<In>>
}
```

---

## shared/testing.ts (Step 3)

```ts
// shared/testing.ts

import type {
  Result, FunctionSpec, FactorySpec, ShellFactorySpec,
} from './spec'

// ── runSpec — atomic functions (sync) ────────────────────────────────────────
// Reads predicates and examples from the spec itself — no separate examples arg.

export const runSpec = <In, Out, F extends string, S extends string>(
  fn:   (input: In) => Result<Out, F, S>,
  spec: FunctionSpec<In, Out, F, S>,
): void => {
  describe('failures', () => {
    for (const [failure, constraint] of Object.entries(spec.constraints) as [F, { predicate: any; examples: { when: In }[] }][]) {
      describe(failure, () => {
        constraint.examples.forEach((example, i) => {
          const label = constraint.examples.length === 1 ? failure : `${failure} [${i + 1}]`
          test(label, () => {
            const result = fn(example.when)
            expect(result.ok).toBe(false)
            if (!result.ok) expect(result.errors).toContain(failure)
          })
        })
      })
    }
  })

  if (spec.mixed && spec.mixed.length > 0) {
    describe('mixed failures', () => {
      for (const example of spec.mixed!) {
        test(example.description, () => {
          const result = fn(example.when)
          expect(result.ok).toBe(false)
          if (!result.ok) {
            for (const f of example.failsWith) {
              expect(result.errors).toContain(f)
            }
          }
        })
      }
    })
  }

  for (const [successType, successSpec] of Object.entries(spec.successes) as [S, any][]) {
    describe(successType, () => {
      successSpec.examples.forEach((example: { when: In; then: Out }, i: number) => {
        const suffix = successSpec.examples.length > 1 ? ` [${i + 1}]` : ''

        test(`condition${suffix}`, () => {
          const result = fn(example.when)
          expect(result.ok).toBe(true)
          if (result.ok)
            expect(successSpec.condition({ input: example.when, output: result.value })).toBe(true)
        })

        for (const [name, assertion] of Object.entries(successSpec.assertions ?? {}) as [string, any][]) {
          test(`${name}${suffix}`, () => {
            const result = fn(example.when)
            expect(result.ok).toBe(true)
            if (result.ok)
              expect(assertion({ input: example.when, output: result.value })).toBe(true)
          })
        }

        test(`expected value${suffix}`, () => {
          const result = fn(example.when)
          expect(result.ok).toBe(true)
          if (result.ok) expect(result.value).toEqual(example.then)
        })

        test(`success type${suffix}`, () => {
          const result = fn(example.when)
          expect(result.ok).toBe(true)
          if (result.ok) expect(result.successType).toContain(successType)
        })
      })
    })
  }
}

// ── runSpecAsync — async atomic functions ────────────────────────────────────

export const runSpecAsync = <In, Out, F extends string, S extends string>(
  fn:   (input: In) => Promise<Result<Out, F, S>>,
  spec: FunctionSpec<In, Out, F, S>,
): void => {
  describe('failures', () => {
    for (const [failure, constraint] of Object.entries(spec.constraints) as [F, { predicate: any; examples: { when: In }[] }][]) {
      describe(failure, () => {
        constraint.examples.forEach((example, i) => {
          const label = constraint.examples.length === 1 ? failure : `${failure} [${i + 1}]`
          test(label, async () => {
            const result = await fn(example.when)
            expect(result.ok).toBe(false)
            if (!result.ok) expect(result.errors).toContain(failure)
          })
        })
      })
    }
  })

  if (spec.mixed && spec.mixed.length > 0) {
    describe('mixed failures', () => {
      for (const example of spec.mixed!) {
        test(example.description, async () => {
          const result = await fn(example.when)
          expect(result.ok).toBe(false)
          if (!result.ok) {
            for (const f of example.failsWith) {
              expect(result.errors).toContain(f)
            }
          }
        })
      }
    })
  }

  for (const [successType, successSpec] of Object.entries(spec.successes) as [S, any][]) {
    describe(successType, () => {
      successSpec.examples.forEach((example: { when: In; then: Out }, i: number) => {
        const suffix = successSpec.examples.length > 1 ? ` [${i + 1}]` : ''

        test(`condition${suffix}`, async () => {
          const result = await fn(example.when)
          expect(result.ok).toBe(true)
          if (result.ok)
            expect(successSpec.condition({ input: example.when, output: result.value })).toBe(true)
        })

        for (const [name, assertion] of Object.entries(successSpec.assertions ?? {}) as [string, any][]) {
          test(`${name}${suffix}`, async () => {
            const result = await fn(example.when)
            expect(result.ok).toBe(true)
            if (result.ok)
              expect(assertion({ input: example.when, output: result.value })).toBe(true)
          })
        }

        test(`expected value${suffix}`, async () => {
          const result = await fn(example.when)
          expect(result.ok).toBe(true)
          if (result.ok) expect(result.value).toEqual(example.then)
        })

        test(`success type${suffix}`, async () => {
          const result = await fn(example.when)
          expect(result.ok).toBe(true)
          if (result.ok) expect(result.successType).toContain(successType)
        })
      })
    })
  }
}

// ── runFactorySpec — core factories (sync) ──────────────────────────────────
// Factories have `failures` (examples only, no predicates) instead of `constraints`.
// No assertions — conditions + expected value match only.

export const runFactorySpec = <In, Out, F extends string, S extends string>(
  fn:   (input: In) => Result<Out, F, S>,
  spec: FactorySpec<In, Out, F, S>,
): void => {
  describe('failures', () => {
    for (const [failure, failureExamples] of Object.entries(spec.failures) as [F, { when: In }[]][]) {
      describe(failure, () => {
        failureExamples.forEach((example, i) => {
          const label = failureExamples.length === 1 ? failure : `${failure} [${i + 1}]`
          test(label, () => {
            const result = fn(example.when)
            expect(result.ok).toBe(false)
            if (!result.ok) expect(result.errors).toContain(failure)
          })
        })
      })
    }
  })

  if (spec.mixed && spec.mixed.length > 0) {
    describe('mixed failures', () => {
      for (const example of spec.mixed!) {
        test(example.description, () => {
          const result = fn(example.when)
          expect(result.ok).toBe(false)
          if (!result.ok) {
            for (const f of example.failsWith) {
              expect(result.errors).toContain(f)
            }
          }
        })
      }
    })
  }

  for (const [successType, successSpec] of Object.entries(spec.successes) as [S, any][]) {
    describe(successType, () => {
      successSpec.examples.forEach((example: { when: In; then: Out }, i: number) => {
        const suffix = successSpec.examples.length > 1 ? ` [${i + 1}]` : ''

        test(`condition${suffix}`, () => {
          const result = fn(example.when)
          expect(result.ok).toBe(true)
          if (result.ok)
            expect(successSpec.condition({ input: example.when, output: result.value })).toBe(true)
        })

        test(`expected value${suffix}`, () => {
          const result = fn(example.when)
          expect(result.ok).toBe(true)
          if (result.ok) expect(result.value).toEqual(example.then)
        })

        test(`success type${suffix}`, () => {
          const result = fn(example.when)
          expect(result.ok).toBe(true)
          if (result.ok) expect(result.successType).toContain(successType)
        })
      })
    })
  }
}

// ── runFactorySpecAsync — async factories ───────────────────────────────────

export const runFactorySpecAsync = <In, Out, F extends string, S extends string>(
  fn:   (input: In) => Promise<Result<Out, F, S>>,
  spec: FactorySpec<In, Out, F, S>,
): void => {
  describe('failures', () => {
    for (const [failure, failureExamples] of Object.entries(spec.failures) as [F, { when: In }[]][]) {
      describe(failure, () => {
        failureExamples.forEach((example, i) => {
          const label = failureExamples.length === 1 ? failure : `${failure} [${i + 1}]`
          test(label, async () => {
            const result = await fn(example.when)
            expect(result.ok).toBe(false)
            if (!result.ok) expect(result.errors).toContain(failure)
          })
        })
      })
    }
  })

  if (spec.mixed && spec.mixed.length > 0) {
    describe('mixed failures', () => {
      for (const example of spec.mixed!) {
        test(example.description, async () => {
          const result = await fn(example.when)
          expect(result.ok).toBe(false)
          if (!result.ok) {
            for (const f of example.failsWith) {
              expect(result.errors).toContain(f)
            }
          }
        })
      }
    })
  }

  for (const [successType, successSpec] of Object.entries(spec.successes) as [S, any][]) {
    describe(successType, () => {
      successSpec.examples.forEach((example: { when: In; then: Out }, i: number) => {
        const suffix = successSpec.examples.length > 1 ? ` [${i + 1}]` : ''

        test(`condition${suffix}`, async () => {
          const result = await fn(example.when)
          expect(result.ok).toBe(true)
          if (result.ok)
            expect(successSpec.condition({ input: example.when, output: result.value })).toBe(true)
        })

        test(`expected value${suffix}`, async () => {
          const result = await fn(example.when)
          expect(result.ok).toBe(true)
          if (result.ok) expect(result.value).toEqual(example.then)
        })

        test(`success type${suffix}`, async () => {
          const result = await fn(example.when)
          expect(result.ok).toBe(true)
          if (result.ok) expect(result.successType).toContain(successType)
        })
      })
    })
  }
}

// ── runShellSpec — shell factories (async + dep propagation) ────────────────
// Reads baseDeps and depPropagation from the spec itself.

export const runShellSpec = <
  In, Out, F extends string, S extends string,
  Deps extends Record<string, any>
>(
  fn:     (input: In) => Promise<Result<Out, F, S>>,
  makeFn: (deps: Deps) => (input: In) => Promise<Result<Out, F, S>>,
  spec:   ShellFactorySpec<In, Out, F, S, Deps>,
): void => {
  runFactorySpecAsync(fn, spec)

  describe('dep propagation', () => {
    for (const [depName, prop] of Object.entries(spec.depPropagation) as [string, { when: In; failsWith: string }][]) {
      test(`propagates ${depName} failure`, async () => {
        const override = { [depName]: async () => ({ ok: false, errors: [prop.failsWith] }) }
        const fn = makeFn({ ...spec.baseDeps, ...override } as Deps)
        const result = await fn(prop.when)
        expect(result.ok).toBe(false)
        if (!result.ok) expect(result.errors).toContain(prop.failsWith)
      })
    }
  })
}
```

---

## scripts/spec-tools.ts (Step 4)

```ts
// scripts/spec-tools.ts

// ── Types ────────────────────────────────────────────────────────────────────

export type FlatConstraint = {
  step:    string    // 'parseCartId', 'checkActive', 'findCartById'
  failure: string    // 'not_a_string', 'cart_empty', 'find_failed'
  type:    'step' | 'dep'
}

export type FlatTable = {
  columns:   FlatConstraint[]
  successes: string[]
}

// ── flattenSpec ──────────────────────────────────────────────────────────────
// Recursively walks factory spec tree, collecting constraints in pipeline order.

export function flattenSpec(spec: any): FlatTable {
  return {
    columns:   flattenConstraints(spec),
    successes: Object.keys(spec.successes),
  }
}

function flattenConstraints(spec: any): FlatConstraint[] {
  const result: FlatConstraint[] = []

  for (const [name, stepSpec] of Object.entries(spec.steps || {}) as any[]) {
    if (stepSpec.steps) {
      // Recursive — this step is itself a factory (e.g. core inside shell)
      result.push(...flattenConstraints(stepSpec))
    } else {
      for (const failure of Object.keys(stepSpec.constraints || {})) {
        result.push({ step: name, failure, type: 'step' })
      }
    }
  }

  for (const [name, depSpec] of Object.entries(spec.deps || {}) as any[]) {
    for (const failure of depSpec.failures) {
      result.push({ step: name, failure, type: 'dep' })
    }
  }

  return result
}

// ── toMarkdownTable ──────────────────────────────────────────────────────────
// Converts flat table to decision table markdown with ✓/✗/— symbols.

export function toMarkdownTable(table: FlatTable): string {
  const { columns, successes } = table

  const header = [
    'Scenario',
    ...columns.map(c => `\`${c.step}\` :${c.failure}`),
    'Outcome',
  ]
  const separator = ['---', ...columns.map(() => ':---:'), '---']

  const successRows = successes.map(s => [
    `✅ ${s}`,
    ...columns.map(() => '✓'),
    s,
  ])

  const failureRows = columns.map((c, i) => [
    `❌ ${c.failure}`,
    ...columns.map((_, j) => j < i ? '✓' : j === i ? '✗' : '—'),
    `Fails: \`${c.failure}\``,
  ])

  const rows = [header, separator, ...successRows, ...failureRows]
  return rows.map(r => `| ${r.join(' | ')} |`).join('\n')
}

// ── toStepTable ──────────────────────────────────────────────────────────────
// Extracts pipeline step table from a factory spec (for §4 and §5 of .spec.md).

export function toStepTable(spec: any): string {
  const rows: string[][] = [
    ['#', 'Name', 'Type', 'Failure Codes'],
    ['---', '---', '---', '---'],
  ]
  let index = 1

  for (const [name, stepSpec] of Object.entries(spec.steps || {}) as any[]) {
    const failures = stepSpec.steps
      ? flattenConstraints(stepSpec).map(c => c.failure)
      : Object.keys(stepSpec.constraints || {})
    const failStr = failures.length > 0 ? failures.map(f => `\`${f}\``).join(', ') : '—'
    rows.push([String(index++), `\`${name}\``, '`STEP`', failStr])
  }

  for (const [name, depSpec] of Object.entries(spec.deps || {}) as any[]) {
    const failStr = depSpec.failures.map((f: string) => `\`${f}\``).join(', ') || '—'
    rows.push([String(index++), `\`${name}\``, '`DEP`', failStr])
  }

  return rows.map(r => `| ${r.join(' | ')} |`).join('\n')
}
```

---

## scripts/spec-manifest.ts (Step 5)

```ts
// scripts/spec-manifest.ts
//
// Add one entry per factory spec. The generate-specs script reads this manifest,
// imports each spec, and writes the .spec.md decision tables next to the spec file.
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

## scripts/generate-specs.ts (Step 6)

```ts
// scripts/generate-specs.ts

import { writeFileSync, readFileSync, existsSync } from 'fs'
import { resolve, dirname, relative } from 'path'
import { flattenSpec, toMarkdownTable, toStepTable } from './spec-tools'
import { specManifest } from './spec-manifest'

const DOCS_SPECS_PATH = resolve(__dirname, '../docs/specs.ts')

const MARKER_BEGIN = (section: string) => `<!-- BEGIN:GENERATED:${section} -->`
const MARKER_END   = (section: string) => `<!-- END:GENERATED:${section} -->`
const STALE_MARKER = '<!-- ⚠ STALE: Generated sections (§4-§5) changed — review this prose section. Run ddd-documentation to update. -->'

function generateSection(tag: string, content: string): string {
  return `${MARKER_BEGIN(tag)}\n${content}\n${MARKER_END(tag)}`
}

function extractGeneratedContent(file: string, tag: string): string {
  const begin = MARKER_BEGIN(tag)
  const end = MARKER_END(tag)
  const regex = new RegExp(
    `${escapeRegex(begin)}\\n([\\s\\S]*?)\\n${escapeRegex(end)}`
  )
  const match = file.match(regex)
  return match ? match[1] : ''
}

function replaceGeneratedSections(existing: string, sections: Record<string, string>): {
  content: string
  changed: boolean
} {
  let result = existing
  let changed = false

  for (const [tag, content] of Object.entries(sections)) {
    const oldContent = extractGeneratedContent(existing, tag)
    if (oldContent !== content) changed = true

    const begin = MARKER_BEGIN(tag)
    const end = MARKER_END(tag)
    const regex = new RegExp(
      `${escapeRegex(begin)}[\\s\\S]*?${escapeRegex(end)}`,
      'g'
    )
    if (result.includes(begin)) {
      result = result.replace(regex, generateSection(tag, content))
    } else {
      result += '\n\n' + generateSection(tag, content)
    }
  }

  return { content: result, changed }
}

// When generated sections change, inject a staleness marker above each
// prose section (§1-§3) so authors know the prose may need updating.
function addStaleMarkers(content: string): string {
  // First remove any existing stale markers to avoid duplication
  let result = removeStaleMarkers(content)

  // Add marker before each prose section heading
  for (const heading of ['## 1. Overview', '## 2. Operation Interface', '## 3. Business Scenarios']) {
    result = result.replace(heading, `${STALE_MARKER}\n\n${heading}`)
  }

  return result
}

function removeStaleMarkers(content: string): string {
  return content.replace(new RegExp(escapeRegex(STALE_MARKER) + '\\n\\n?', 'g'), '')
}

function escapeRegex(s: string): string {
  return s.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')
}

function newSpecMd(name: string, sections: Record<string, string>): string {
  return `# ${name}

> **Operation Specification**

---

## 1. Overview

<!-- TODO: run ddd-documentation to fill this section -->

---

## 2. Operation Interface

<!-- TODO: run ddd-documentation to fill this section -->

---

## 3. Business Scenarios

<!-- TODO: run ddd-documentation to fill this section -->

---

## 4. Pipeline

${generateSection('PIPELINE', sections.PIPELINE)}

---

## 5. Decision Tables

${generateSection('DECISION', sections.DECISION)}
`
}

// ── Manifest validation ──────────────────────────────────────────────────────
// Checks every manifest entry points to a real file with the expected export.
// Runs before any generation — catches stale entries from renames or deletions.

function validateManifest(): { valid: typeof specManifest; errors: string[] } {
  const valid: typeof specManifest = []
  const errors: string[] = []

  for (const entry of specManifest) {
    let resolvedPath: string
    try {
      resolvedPath = require.resolve(entry.specPath)
    } catch {
      errors.push(`✗ ${entry.name}: file not found — ${entry.specPath}`)
      continue
    }

    // eslint-disable-next-line @typescript-eslint/no-var-requires
    const mod = require(resolvedPath)
    if (!mod[entry.exportName]) {
      errors.push(`✗ ${entry.name}: export '${entry.exportName}' not found in ${resolvedPath}`)
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

  // Validate manifest before doing any work
  const { valid: validEntries, errors: manifestErrors } = validateManifest()
  if (manifestErrors.length > 0) {
    console.error('\nManifest validation failed:')
    manifestErrors.forEach(e => console.error(`  ${e}`))
    console.error(`\nFix the manifest (scripts/spec-manifest.ts) and re-run.\n`)
    process.exit(1)
  }

  const generatedPaths: Record<string, string> = {}

  for (const entry of validEntries) {
    const mod = await import(entry.specPath)
    const spec = mod[entry.exportName]

    const table = flattenSpec(spec)
    const pipelineContent = toStepTable(spec)
    const decisionContent = toMarkdownTable(table)

    const sections = {
      PIPELINE: pipelineContent,
      DECISION: decisionContent,
    }

    // Resolve output path: same dir as spec, .spec.md extension
    const specFile = require.resolve(entry.specPath)
    const mdPath = specFile.replace(/\.spec\.ts$/, '.spec.md')

    if (existsSync(mdPath)) {
      const existing = readFileSync(mdPath, 'utf-8')
      const { content: updated, changed } = replaceGeneratedSections(existing, sections)

      if (changed) {
        // Generated content changed — mark prose sections as potentially stale
        const withMarkers = addStaleMarkers(updated)
        writeFileSync(mdPath, withMarkers)
        console.log(`✓ ${entry.name}: updated generated sections in ${mdPath}`)
        console.log(`  ⚠ Prose sections (§1-§3) may be stale — run ddd-documentation to review`)
      } else {
        console.log(`· ${entry.name}: no changes`)
      }
    } else {
      const content = newSpecMd(entry.name, sections)
      writeFileSync(mdPath, content)
      console.log(`✓ ${entry.name}: created ${mdPath}`)
    }

    // Track for docs/specs.ts — path relative to project root
    const projectRoot = resolve(__dirname, '..')
    generatedPaths[entry.name] = relative(projectRoot, mdPath)
  }

  // Update docs/specs.ts — the documentation registry
  const entries = Object.entries(generatedPaths)
    .map(([name, path]) => `  '${name}': '${path}',`)
    .join('\n')
  const docsContent = `// docs/specs.ts
//
// Documentation registry — lists all generated .spec.md files.
// Auto-updated by \`npm run gen:specs\`. Do not edit manually.

export const specDocs: Record<string, string> = {
${entries}
}
`
  writeFileSync(DOCS_SPECS_PATH, docsContent)
  console.log(`✓ docs/specs.ts: updated with ${Object.keys(generatedPaths).length} entries`)
}

main().catch(err => {
  console.error('generate-specs failed:', err)
  process.exit(1)
})
```

---

## scripts/tsconfig.json (Step 7)

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

## docs/specs.ts (Step 8)

```ts
// docs/specs.ts
//
// Documentation registry — lists all generated .spec.md files.
// Auto-updated by `npm run gen:specs`. Do not edit manually.

export const specDocs: Record<string, string> = {
  // Populated automatically from the spec manifest
}
```
