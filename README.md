# ddd-skills

A set of Claude Agent Skills for building intentional, tested TypeScript
using domain-driven design patterns. Adapted from *Domain Modelling Made
Functional* and designed for AI-assisted development.

These skills guide you through a full development pipeline — from data model
to tested factory — one decision at a time, with validation at every step.

---

## The pipeline

```
ddd-data-modelling      Design domain types, primitives, failure unions
      ↓
ddd-examples            Build example declarations per function
      ↓                 (validInputs, invalidInputs, conversions)
ddd-test-suite          Generate *.test.ts files from examples
      ↓
ddd-implement           Implement functions until all tests are green
      ↓
ddd-factory             Decompose a function into steps + factory body
      ↓
ddd-factory-examples    Compose FactoryFailure, generate propagation scenarios
      ↓
ddd-factory-test-suite  Generate *.factory.test.ts files
```

Each skill has a single job. Each skill's output is the next skill's input.
`ARCHITECTURE.md` defines the canonical folder structure all skills conform to.

---

## Requirements

- Claude Pro, Max, Team, or Enterprise plan
- Code execution enabled (Settings → Features → Code execution and file creation)
- For Claude Code: any plan with Claude Code access

---

## Install

### Claude Code (recommended)

**Personal install** — skills available across all your projects:

```bash
git clone https://github.com/your-org/ddd-skills
cp -r ddd-skills/skills/* ~/.claude/skills/
```

**Project install** — skills scoped to one repo:

```bash
cd your-project
git clone https://github.com/your-org/ddd-skills
cp -r ddd-skills/skills/* .claude/skills/
# or add as a git submodule
git submodule add https://github.com/your-org/ddd-skills .claude/ddd-skills
cp -r .claude/ddd-skills/skills/* .claude/skills/
```

Claude Code discovers skills automatically. No configuration needed.

---

### Claude.ai

Each skill must be uploaded individually as a zip file.

1. Go to **Settings → Features → Custom Skills**
2. For each skill folder under `skills/`:
   - Zip the folder (e.g. `ddd-data-modelling.zip` containing `SKILL.md`)
   - Click **Upload skill** and select the zip
3. Repeat for all seven skills
4. Ensure **Code execution and file creation** is enabled in Settings → Features

> Note: Claude.ai skills are per-user. Each colleague uploads separately.

---

### Quick install script (Claude Code)

```bash
#!/bin/bash
# install-ddd-skills.sh
# Run from the cloned ddd-skills directory

SKILLS_DIR="${1:-$HOME/.claude/skills}"
mkdir -p "$SKILLS_DIR"

for skill in skills/*/; do
  name=$(basename "$skill")
  mkdir -p "$SKILLS_DIR/$name"
  cp "$skill/SKILL.md" "$SKILLS_DIR/$name/SKILL.md"
  echo "installed: $name"
done

echo "Done. $SKILLS_DIR now contains all ddd-skills."
```

```bash
chmod +x install-ddd-skills.sh
./install-ddd-skills.sh                        # installs to ~/.claude/skills/
./install-ddd-skills.sh .claude/skills/        # installs to current project
```

---

## Usage

Skills are invoked automatically by Claude when relevant, or explicitly:

```
# In Claude Code or Claude.ai
"help me model the order domain"              → triggers ddd-data-modelling
"build examples for parseUsername"            → triggers ddd-examples
"generate the test file for username"         → triggers ddd-test-suite
"implement parseUsername"                     → triggers ddd-implement
"help me build the confirmOrder factory"      → triggers ddd-factory
"build factory examples for confirmOrder"     → triggers ddd-factory-examples
"generate the factory test for confirmOrder"  → triggers ddd-factory-test-suite
```

You can also reference skills explicitly:
```
"use ddd-data-modelling to help me design the payment domain"
```

---

## What you get

After running the full pipeline for one aggregate, you have:

```
src/
├── shared/
│   └── testing.ts              ← runExamples, runFactoryExamples, all types
├── {aggregate}/
│   ├── types.ts                ← domain types + failure unions inline
│   ├── parsing/
│   │   ├── {primitive}.ts      ← parse function
│   │   ├── {primitive}.examples.ts
│   │   ├── {primitive}.examples.md   ← plain-English spec for PMs
│   │   └── {primitive}.test.ts
│   ├── shared/                 ← reusable steps
│   └── service/
│       └── {factory}/
│           ├── {factory}.ts
│           ├── {factory}.factory-examples.ts
│           ├── {factory}.factory.test.ts
│           └── steps/
├── infra/                      ← dep implementations (not generated)
└── app/index.ts                ← wiring (not generated)
```

See `ARCHITECTURE.md` for the full structure and import rules.

---

## Key concepts

**Failure unions as specs**
```ts
type UsernameFailure =
  | 'not_a_string'
  | 'empty'
  | 'too_short_min_3'
  | 'too_long_max_20'
  | 'invalid_chars_alphanumeric_and_underscores_only'
  | 'reserved_word'
  | 'script_injection'
// Reading this type tells you the complete rulebook. No docs needed.
```

**Examples as behavioural contracts**
```ts
const usernameInvalidInputs = {
  too_short_and_invalid: {
    input:     'a!',
    failsWith: ['too_short_min_3', 'invalid_chars_alphanumeric_and_underscores_only'],
  },
} satisfies InvalidInputs<UsernameFailure>
// Plain objects. Reviewable by PMs. Type-safe. AI-generatable.
```

**Factory as algorithm**
```ts
const confirmOrderFactory = (steps: Steps) => (deps: Deps) => async (
  orderId: OrderId
): Promise<Result<ConfirmedOrder>> => {
  // 1. fetch the order by id from persistence
  // 2. check the order is in pending state       ← pure step
  // 3. validate product ids via external service
  // 4. build the confirmed order                 ← pure step
  // 5. save the confirmed order to persistence
  // 6. log success
}
// The comments are the spec. The types enforce the contract.
```

---

## Feedback

These skills are under active development. If something doesn't work as
expected or a skill produces output that doesn't match the architecture:

- Open an issue on GitHub
- Or note which skill, which step, and what it produced vs what you expected

The skills work best when you slow down at validation gates — especially
failure unions and mixed examples. That's where your domain knowledge matters
most and where the AI is most likely to need correction.

---

## Skill summary

| Skill | Input | Output | Key validation gate |
|-------|-------|--------|-------------------|
| `ddd-data-modelling` | domain description | `types.ts` | failure union per primitive |
| `ddd-examples` | type + failure union | `*.examples.ts` + `*.examples.md` | mixed dirty inputs |
| `ddd-test-suite` | `*.examples.ts` | `*.test.ts` + `shared/testing.ts` | red suite confirmed |
| `ddd-implement` | failing test suite | green `*.ts` | all tests pass |
| `ddd-factory` | function signature | factory scaffold + step queue | algorithm as comments |
| `ddd-factory-examples` | factory + step failures | `*.factory-examples.ts` | FactoryFailure composition |
| `ddd-factory-test-suite` | `*.factory-examples.ts` | `*.factory.test.ts` | red suite confirmed |
