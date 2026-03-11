# DDD Skills Pipeline

A set of Claude Code skills for building domain-driven TypeScript applications.
Each skill handles one phase of the design-to-implementation pipeline — from
domain discovery to green tests.

## The pipeline

```
ddd-discover          What are we building? Aggregates, commands, policies.
    ↓                 Output: domain.md
ddd-data-modelling    Types for one aggregate. Lifecycle states, value objects, failure unions.
    ↓                 Output: types.ts
ddd-spec              Behavioral contract. Constraints, conditions, assertions, examples.
    ↓                 Output: *.spec.ts
ddd-test-suite        Wire spec to runners. Minimal test file.
    ↓                 Output: *.test.ts
ddd-implement         Simplest code that makes tests green.
    ↓                 Output: *.ts
ddd-documentation     Business-friendly spec from decision tables.
                      Output: *.spec.md
```

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

## Quick start

1. **Set up infrastructure** — Run `ddd-init` to create shared types, test
   runners, CLI scripts, and automation hooks.

2. **Discover the domain** — Run `ddd-discover` to identify aggregates,
   their responsibilities, and how they interact. Produces `domain.md`.

3. **Pick an aggregate and model it** — Run `ddd-data-modelling` to design
   types for one aggregate. Produces `types.ts`.

4. **Build functions inside-out** — For each function in the aggregate,
   run the spec → test-suite → implement pipeline.

## Skill overview

| Skill | Input | Output | Key idea |
|---|---|---|---|
| **ddd-discover** | Conversation | `domain.md` | Aggregates, commands, policies. No code. |
| **ddd-init** | — | Shared infra files | `spec.ts`, `testing.ts`, CLI scripts, hooks. Run once. |
| **ddd-data-modelling** | `domain.md` or conversation | `types.ts` | Discriminated unions, value objects, failure unions. |
| **ddd-spec** | `types.ts` + function description | `*.spec.ts` | Predicates AND examples unified. Constraints, conditions, assertions, concrete data. |
| **ddd-test-suite** | `*.spec.ts` | `*.test.ts` | Imports + one runner call. No logic. |
| **ddd-implement** | `*.spec.ts` + `*.test.ts` | `*.ts` | Simplest code that passes. Parse accumulates, factory short-circuits. |
| **ddd-documentation** | `*.spec.ts` + CLI output | `*.spec.md` | Business-friendly markdown with decision tables. |

## Function taxonomy

Every function is one of three types:

- **Step** — Pure, sync, single-concern. Guards, transforms, parses. Atomic
  building block. Tested with `runSpec`.
- **Core factory** — Pure, sync, orchestrates steps. No I/O, no deps. Tested
  with `runSpec` using real steps.
- **Shell factory** — Async, bridges infra with domain. Parses input, calls
  deps, calls core, persists. Tested with `runShellSpec` (adds dep propagation).

Steps are built first, then composed into core, then wrapped in shell.

## Spec types

- **FunctionSpec** — For atomic functions. Has constraints (what can fail) and
  successes with conditions and assertions (what's true about each outcome).
- **FactorySpec** — For core factories. No constraint predicates (inherited from
  step specs), has failure examples and success conditions with examples.
- **ShellFactorySpec** — Extends FactorySpec with `baseDeps` and `depPropagation`.

## Folder structure

```
src/
  shared/
    spec.ts                   ← Result, FunctionSpec, FactorySpec, ShellFactorySpec
    testing.ts                ← runSpec, runSpecAsync, runFactorySpec, runShellSpec

  cart/                       ← one folder per aggregate
    types.ts                  ← domain types, primitives, failure unions
    shared/steps/             ← reusable steps across operations
    subtract-quantity/        ← one folder per operation
      *.spec.ts / *.test.ts / *.ts / *.spec.md
      core/                   ← pure domain logic
        *.spec.ts / *.test.ts / *.ts

scripts/                      ← CLI tools for spec generation
docs/                         ← documentation registry
```

## Skill file structure

Each skill directory may contain:

| File | Purpose | Loaded |
|---|---|---|
| `SKILL.md` | Workflow, disposition, hard rules | Always |
| `reference.md` | Detailed conventions and API docs | When needed |
| `examples.md` | Complete code examples | When needed |

`SKILL.md` is kept concise. Large code blocks live in companion files to
keep the primary instructions focused on workflow and decision-making.

## Conventions

- **Files:** kebab-case — `check-active.spec.ts`
- **Exports:** camelCase — `checkActiveSpec`
- **Failures:** snake_case — `cart_empty`, `not_a_string`
- **Success types:** kebab-case, **past tense** — domain events describing what
  happened. `cart-id-parsed`, `quantity-reduced`, `cart-emptied`
- **Single input object** for functions with >1 parameter
- **No classes, no frameworks.** Plain types and pure functions only.
- **No brands.** Plain type aliases. Meaning comes from parsing at boundaries.
- **Errors never throw.** All failures are `Result<T, F, S>`.

## Automation and `.spec.md` lifecycle

Each `.spec.md` file has **two owners**:

| Sections | Owner | Updated by |
|---|---|---|
| §1-§3 (Overview, Interface, Scenarios) | AI (`ddd-documentation`) | Running the skill manually |
| §4-§5 (Pipeline, Decision Tables) | CLI (`gen:specs`) | Automatically on `.spec.ts` save |

**The flow:**
1. You edit a `.spec.ts` file (add a constraint, change a success type)
2. The Claude Code hook runs `npm run gen:specs` automatically
3. The CLI updates §4-§5 between `<!-- BEGIN:GENERATED -->` markers
4. If the generated content actually changed, the CLI injects
   `<!-- ⚠ STALE -->` markers above §1-§3, signaling the prose may need review
5. Running `ddd-documentation` updates the prose and removes the stale markers

The spec manifest (`scripts/spec-manifest.ts`) is updated by `ddd-spec`
when creating factory specs.

## Design principles

**Decision tables are the primary spec artifact.** The `.spec.ts` file with
typed predicates is the source of truth. Markdown documentation is derived.

**Compiler enforces exhaustiveness.** `Record<F, ...>` in specs means you can't
forget a failure case. Add a constraint to the spec → compiler error until the
example exists.

**Each skill does one thing.** No skill both designs and implements. The handoff
points are explicit: spec → tests → implementation.

**Inside-out implementation.** Atomic steps first, then core factory, then shell.
Each layer's tests pass before the next layer depends on it.

**Conversation over automation.** Skills guide through decisions one at a time.
They propose, explain briefly, and wait for confirmation. The human makes the
domain decisions — the skill handles the mechanical consistency.
