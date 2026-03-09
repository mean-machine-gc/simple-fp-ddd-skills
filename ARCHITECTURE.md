# ARCHITECTURE.md
# Canonical folder structure and file naming conventions.
# All ddd-* skills read this file before producing any file path.
# No skill may produce a path that contradicts this document.

---

## Guiding principles

1. **Files stack by name.** `username.ts`, `username.examples.ts`,
   `username.examples.md`, `username.test.ts` share a prefix and group
   naturally in any file explorer. No separate folders for examples or tests
   within a concern.

2. **Aggregate boundaries are folder boundaries.** Each aggregate has one
   root folder. Types, parsing, steps, and factories for that aggregate live
   inside it. Nothing crosses aggregate folders except through explicit imports.

3. **Domain code never imports from infra.** Types, parsing, steps, and
   factories import only from `shared/testing.ts` and sibling files within
   the aggregate. Infrastructure imports from the aggregate — never the
   reverse.

4. **The aggregate declares capabilities, infra implements them.** `Deps`
   types live inside the aggregate as function signatures. The implementations
   live in `infra/` and are wired at `app/`.

5. **`shared/testing.ts` is project-wide.** It is written once by
   `ddd-test-suite` and never modified by other skills. All test files
   import from it.

---

## Full project structure

```
src/
├── shared/
│   └── testing.ts                  ← Result, Example, FactoryExample,
│                                      buildExamples, identityConversions,
│                                      runExamples, runExamplesAsync,
│                                      runFactoryExamples, runFactoryExamplesAsync
│
├── {aggregate}/                    ← one folder per aggregate (e.g. order/, user/)
│   ├── types.ts                    ← all domain types for this aggregate:
│   │                                  primitives, failure unions (inline),
│   │                                  value objects, entity lifecycle types,
│   │                                  Result<T>, Parse<T>, Steps/Deps/Ctx shapes
│   │
│   ├── parsing/                    ← one file-set per parse function
│   │   ├── {primitive}.ts          ← parse function implementation
│   │   ├── {primitive}.examples.ts ← validInputs, invalidInputs, examples array
│   │   ├── {primitive}.examples.md ← plain-English spec for non-technical review
│   │   └── {primitive}.test.ts     ← runExamples call only
│   │
│   ├── shared/                     ← steps reused across multiple factories
│   │   ├── {step}.ts
│   │   ├── {step}.examples.ts
│   │   ├── {step}.examples.md
│   │   └── {step}.test.ts
│   │
│   │   ── choose ONE of: service/ OR shell/ + core/ ──────────────────────────
│   │
│   ├── service/                    ← async factories (variant 3)
│   │   └── {factory}/
│   │       ├── {factory}.ts                      ← factory implementation
│   │       ├── {factory}.factory-examples.ts     ← FactoryFailure, baseDeps,
│   │       │                                        realSteps, factory examples
│   │       ├── {factory}.factory.test.ts         ← runFactoryExamplesAsync call
│   │       └── steps/                            ← steps local to this factory
│   │           ├── {step}.ts
│   │           ├── {step}.examples.ts
│   │           ├── {step}.examples.md
│   │           └── {step}.test.ts
│   │
│   ├── core/                       ← pure sync factories (variant 2)
│   │   └── {factory}/
│   │       ├── {factory}.ts
│   │       ├── {factory}.factory-examples.ts
│   │       ├── {factory}.factory.test.ts
│   │       └── steps/
│   │           ├── {step}.ts
│   │           ├── {step}.examples.ts
│   │           ├── {step}.examples.md
│   │           └── {step}.test.ts
│   │
│   └── shell/                      ← async shell factories (variant 4)
│       └── {factory}/
│           ├── {factory}.ts                      ← shell factory implementation
│           ├── {factory}.factory-examples.ts
│           ├── {factory}.factory.test.ts
│           └── steps/                            ← shell steps (buildCtx etc.)
│               ├── {step}.ts
│               ├── {step}.examples.ts
│               ├── {step}.examples.md
│               └── {step}.test.ts
│
├── infra/                          ← dep implementations — not domain code
│   ├── {aggregate}-repo.ts         ← implements persistence deps for aggregate
│   └── {service}-client.ts         ← implements external service deps
│
└── app/
    └── index.ts                    ← wires infra into factories, exports bound fns
```

---

## Concrete example — order aggregate, service variant

```
src/
├── shared/
│   └── testing.ts
│
├── order/
│   ├── types.ts
│   │     type OrderId = string
│   │     type OrderIdFailure = 'not_a_string' | 'not_a_uuid'
│   │     type UnconfirmedOrder = { status: 'pending'; ... }
│   │     type ConfirmedOrder   = { status: 'confirmed'; ... }
│   │
│   ├── parsing/
│   │   ├── order-id.ts
│   │   ├── order-id.examples.ts
│   │   ├── order-id.examples.md
│   │   └── order-id.test.ts
│   │
│   ├── shared/
│   │   ├── check-pending.ts
│   │   ├── check-pending.examples.ts
│   │   ├── check-pending.examples.md
│   │   └── check-pending.test.ts
│   │
│   └── service/
│       └── confirm-order/
│           ├── confirm-order.ts
│           ├── confirm-order.factory-examples.ts
│           ├── confirm-order.factory.test.ts
│           └── steps/
│               ├── build-confirmed.ts
│               ├── build-confirmed.examples.ts
│               ├── build-confirmed.examples.md
│               └── build-confirmed.test.ts
│
├── infra/
│   ├── order-repo.ts               ← findOrderById, saveOrder
│   └── product-service-client.ts   ← validateProductIds
│
└── app/
    └── index.ts
```

---

## Import rules

Every file may only import from:

| File type          | May import from                                      |
|--------------------|------------------------------------------------------|
| `types.ts`         | external type libraries only                         |
| `parsing/*.ts`     | `../../shared/testing`, `../types`                   |
| `parsing/*.examples.ts` | `../../shared/testing`, `../types`              |
| `parsing/*.test.ts`| `../../shared/testing`, `./{primitive}.examples`, `./{primitive}` |
| `shared/*.ts`      | `../../shared/testing`, `../types`                   |
| `service/{f}/{f}.ts` | `../../../shared/testing`, `../../types`, `./steps/*`, `../../shared/*` |
| `service/{f}/steps/*.ts` | `../../../shared/testing`, `../../../types`   |
| `infra/*.ts`       | `../order/types` (to satisfy Deps signatures)        |
| `app/index.ts`     | everything — this is the wiring layer                |

**Never:**
- `types.ts` importing from `parsing/`, `steps/`, or `infra/`
- `parsing/` or `steps/` importing from `infra/`
- `infra/` importing from `service/`, `shell/`, or `core/`
- Cross-aggregate imports anywhere except `app/index.ts`

---

## Naming conventions

| Thing                        | Convention                        | Example                        |
|------------------------------|-----------------------------------|--------------------------------|
| Aggregate folder             | kebab-case noun                   | `order/`, `user/`, `product/`  |
| Type file                    | `types.ts`                        | `order/types.ts`               |
| Parse function               | `parse-{primitive}.ts`            | `parse-order-id.ts`            |
| Step function file           | `{verb}-{noun}.ts`                | `check-pending.ts`             |
| Factory file                 | `{verb}-{noun}.ts`                | `confirm-order.ts`             |
| Examples file                | `{name}.examples.ts`              | `check-pending.examples.ts`    |
| Markdown spec                | `{name}.examples.md`              | `check-pending.examples.md`    |
| Test file                    | `{name}.test.ts`                  | `check-pending.test.ts`        |
| Factory examples file        | `{name}.factory-examples.ts`      | `confirm-order.factory-examples.ts` |
| Factory test file            | `{name}.factory.test.ts`          | `confirm-order.factory.test.ts` |
| Dep failure literal          | `'{dep_fn_name}_failed'`          | `'find_order_by_id_failed'`    |
| Failure union type           | `{Type}Failure`                   | `UsernameFailure`              |
| Factory failure union        | `{Factory}Failure`                | `ConfirmOrderFailure`          |

---

## Dependency arrow

```
app/index.ts
    │
    ├──▶ infra/                 (implements Deps)
    │         └──▶ {agg}/types  (to satisfy Deps signatures)
    │
    └──▶ {agg}/service|shell    (wires deps into factory)
              └──▶ {agg}/types
              └──▶ {agg}/shared
              └──▶ {agg}/service/{f}/steps

shared/testing.ts ◀── everything test-related
```

No arrow ever points toward `app/` or `infra/` from domain code.
