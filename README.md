# DDD Skills v4

A pipeline of AI skills for building domain-driven TypeScript applications
with typed behavioral contracts.

## Install

### Claude Code (recommended)

**Personal install** — skills available across all your projects:

```bash
git clone https://github.com/mean-machine-gc/simple-fp-ddd-skills
cp -r simple-fp-ddd-skills/skills/* ~/.claude/skills/
```

**Project install** — skills scoped to one repo:

```bash
cd your-project
git clone https://github.com/mean-machine-gc/simple-fp-ddd-skills
cp -r simple-fp-ddd-skills/skills/* .claude/skills/
# or add as a git submodule
git submodule add https://github.com/mean-machine-gc/simple-fp-ddd-skills .claude/simple-fp-ddd-skills
cp -r .claude/simple-fp-ddd-skills/skills/* .claude/skills/
```

Claude Code discovers skills automatically. No configuration needed.

`ddd-init` runs once before any other skill to set up shared infrastructure.

## The pipeline

```
ddd-init ──> ddd-data-modelling ──> ddd-spec ──> ddd-test-suite ──> ddd-implement ──> ddd-documentation
   │                │                   │              │                  │                   │
   │           types.ts            .spec.ts        .test.ts            .ts            docs/*.md
   │                                    │                                        (Jekyll Just the Docs)
   └─ spec-framework.ts            SpecFn + Spec<Fn>
      scripts/                     + steps array
      docs/_config.yml
```

## Core idea

Every function — parse, step, core factory, shell factory — gets the same treatment:

1. **SpecFn** — declares the function contract (input, output, failures, success types)
2. **Spec** — declares the behavioral contract (what fails, what succeeds, what's asserted)
3. **testSpec()** — universal runner, generates tests from the spec
4. **Implementation** — typed via `Fn['signature']` or `Fn['asyncSignature']`

One type (`Spec<Fn>`) for everything. One runner (`testSpec`) for everything.
One test file pattern (4 lines) for everything.

## Key types

```ts
SpecFn<I, O, F, S>     — function contract bundle (signature, asyncSignature, input, output, failures, successTypes)
Spec<Fn>               — behavioral contract (shouldFailWith, shouldSucceedWith, shouldAssert, steps?, document?)
StepInfo               — algorithm step (name, type, description, spec?)
StrategyFn<N,I,O,C,F,S> — strategy dispatch contract (name, input, output, cases, failures, successTypes, handlers)
CanonicalFn<Fn>        — standardized implementation (constraints → conditions → transform), executed via execCanonical
asStepSpec(spec)       — absorbs AnyFn erasure cast for StepInfo.spec and StrategyStep.handlers
Result<T, F, S>        — ok with value + successType, or errors
```

## What's new in v4

| v3 | v4 |
|---|---|
| `FunctionSpec` + `FactorySpec` + `ShellFactorySpec` | Single `Spec<Fn>` |
| `runSpec` + `runFactorySpec` + `runShellSpec` | Single `testSpec()` |
| `spec.ts` + `testing.ts` | Single `spec-framework.ts` |
| Constraint predicates in spec | Spec is pure data + assertions |
| Manual `inherit()` calls | Auto-inheritance via `steps` + `inheritFromSteps()` |
| Separate factory skill for algorithm | `steps` array in spec |
| Manual strategy `coveredBy` | Auto-inherited via `StrategyStep.handlers` |
| 4 type params on spec | `SpecFn<I,O,F,S>` + `Spec<Fn>` (1 param) |
| `when` / `then` | `whenInput` / `then` + `description` |
| No formal factory return type | `Fn['signature']` / `Fn['asyncSignature']` |
| Prose + tables in same `.spec.md` (two owners) | Structural `.spec.md` (CLI) + prose in `/docs/` (Jekyll Just the Docs) |
| Manual spec manifest | Auto-discovery via `document: true` on spec |
| `as unknown as Spec<AnyFn>` everywhere | `asStepSpec()` helper absorbs the cast |
| Shell factories untested | Shell test files with inline mock deps |

## Skills

| Skill | Purpose | Input | Output |
|---|---|---|---|
| **ddd-init** | Project infrastructure setup | — | `spec-framework.ts`, scripts, hooks |
| **ddd-data-modelling** | Domain type design | Domain description | `types.ts` |
| **ddd-spec** | Behavioral contract design | `types.ts` + function description | `.spec.ts` |
| **ddd-test-suite** | Test file generation | `.spec.ts` | `.test.ts` (4 lines) |
| **ddd-implement** | Implementation | `.spec.ts` + `.test.ts` | `.ts` |
| **ddd-documentation** | Business-friendly docs | `.spec.ts` | `docs/<aggregate>/<operation>.md` (Jekyll) |

## Conventions

See [ddd-init/reference.md](ddd-init/reference.md) for the full reference:
- Folder structure, file naming, naming conventions
- Function taxonomy (step, core factory, shell factory)
- Standard input shapes (shell = cmd, core = {cmd, state, ctx})
- Spec structure, testing approach, implementation typing
