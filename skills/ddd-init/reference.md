# DDD Project Conventions

## Folder Structure

Operation-first layout. Each operation gets its own folder. Shell at parent level,
core in `core/` subfolder. Shared steps live in `shared/steps/`.

```
src/
  shared/
    spec.ts                 ← Result, FunctionSpec, FactorySpec, ShellFactorySpec, etc.
    testing.ts              ← runSpec, runFactorySpec, runShellSpec

  cart/
    types.ts                ← domain types, primitives, failure unions

    shared/steps/           ← reusable atomic steps (used across operations)
      check-active.spec.ts
      check-active.test.ts
      check-active.ts

    subtract-quantity/      ← one folder per operation
      subtract-quantity.spec.ts       ← shell spec (predicates + examples)
      subtract-quantity.spec.md       ← documentation (CLI-generated + AI prose)
      subtract-quantity.test.ts
      subtract-quantity.ts            ← shell factory implementation
      core/
        subtract-quantity.spec.ts     ← core spec (predicates + examples)
        subtract-quantity.spec.md
        subtract-quantity.test.ts
        subtract-quantity.ts          ← core factory implementation

scripts/
  spec-tools.ts             ← flattenSpec, toMarkdownTable, toStepTable
  spec-manifest.ts          ← registry of factory specs
  generate-specs.ts         ← entry point: reads manifest, writes .spec.md
  tsconfig.json

docs/
  specs.ts                  ← documentation registry

.claude/
  hooks.json                ← PostToolUse hook for .spec.ts auto-regeneration
```

## File Naming

Every function produces up to 4 co-located files:

| File | Purpose | Created by |
|---|---|---|
| `name.spec.ts` | Behavioral contract: predicates + concrete examples | ddd-spec |
| `name.test.ts` | Test runner wiring (imports + one call) | ddd-test-suite |
| `name.ts` | Implementation | ddd-implement |
| `name.spec.md` | Business-friendly documentation | CLI + ddd-documentation |

All files live in the same directory. No separation by concern (no `tests/` folder).

## Naming Conventions

- **Files:** kebab-case — `check-active.spec.ts`, `subtract-quantity.ts`
- **Exports:** camelCase — `checkActiveSpec`, `subtractQuantityShellFactory`
- **Failure literals:** snake_case — `cart_empty`, `not_a_string`
- **Success types:** kebab-case, **past tense** — domain events describing what
  happened. `cart-id-parsed`, `quantity-reduced`, `cart-emptied`. Never present
  tense (`cart-is-active`) or noun phrases (`cart-total`)
- **Assertion names:** kebab-case — `status-is-active`, `total-recalculated`

## Single Input Object

Functions and factories with more than one parameter always take a single object:

```ts
// YES
type CoreInput = { cart: ActiveCart; productId: ProductId; quantity: Quantity }
const subtractQuantityCore = (input: CoreInput) => Result<...>

// NO — separate args
const subtractQuantityCore = (cart: ActiveCart, productId: ProductId, quantity: Quantity) => Result<...>
```

This enables clean scenario testing — one `when` field covers the full input.

## Shell / Core Split

- **Shell** — async, has deps (persistence, external services). Lives at the
  operation folder root: `subtract-quantity/subtract-quantity.ts`
- **Core** — pure, sync, no deps. Lives in `core/` subfolder:
  `subtract-quantity/core/subtract-quantity.ts`
- **Shared steps** — pure atomic functions reused across operations. Live in
  `shared/steps/`: `shared/steps/check-active.ts`

Shell calls core as a step. Core composes atomic steps. Atomic steps are leaf nodes.

## Function Taxonomy

```
Shell (async, has deps)
  → bridges app/infra with domain
  → parses input (steps), resolves context (deps), calls core (step), persists (deps)
  → only place async and I/O exist

Core (sync, pure, no deps)
  → implements core domain logic of an operation
  → everything from outside (persistence, context) provided by shell
  → orchestrates domain steps

Step (sync, pure, atomic)
  → single-concern function (guard, transform, parse)
  → can itself be a factory of smaller steps (recursive)
  → includes parse functions — they're just steps that take unknown input
```

No async domain functions. If it's async, it touches I/O — it's shell.

## Spec Types

- **Atomic functions** use `FunctionSpec` — has constraints (predicate + examples)
  and successes (condition + assertions + examples). The rule and its proof together.
- **Factories** use `FactorySpec` — no constraint predicates (inherited from step
  specs), but has `failures` (examples only). Successes have conditions + examples
  (no assertions). References step specs as values.
- **Shell factories** use `ShellFactorySpec` — extends `FactorySpec` with
  `baseDeps` and `depPropagation`.

## Spec Predicate Reuse

Implementations should import and use spec constraint predicates by default —
not rewrite the same logic independently. This keeps spec and implementation
in sync: when a predicate changes, the implementation changes with it.

Write logic directly only when the predicate doesn't fit (e.g. needs capture
groups or intermediate values). Comment why when you do.

## Testing Approach

- **Atomic functions** — tested with `runSpec(fn, spec)`. Spec contains both
  predicates and examples.
- **Core factories** — tested with `runFactorySpec(fn, spec)`. Real steps, real
  data, no overrides. Wiring tested end-to-end.
- **Shell factories** — tested with `runShellSpec(fn, makeFn, spec)`. Reads
  `baseDeps` and `depPropagation` from the spec. Step logic proven at step level.
- **Test files are minimal** — imports and one runner call. No logic, no mocks,
  no custom assertions.

## Spec Manifest

Factory specs are registered in `scripts/spec-manifest.ts`. One entry per factory.
The `ddd-spec` skill adds entries when creating factory specs. Atomic function specs
are not registered — they don't have step trees to flatten.

The manifest drives `npm run gen:specs`, which produces `.spec.md` files with
decision tables. A Claude Code hook auto-runs this when `.spec.ts` files change.
