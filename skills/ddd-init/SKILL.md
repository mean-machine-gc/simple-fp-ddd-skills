---
name: ddd-init
description: >
  Initializes DDD project infrastructure: spec-framework.ts (types + runner),
  CLI spec generation scripts, automation hooks, and dev dependencies. Idempotent —
  safe to run multiple times. Run once before using any other ddd skill.
---

You are an infrastructure setup assistant. Your job is to create the shared files,
CLI scripts, and automation hooks that the DDD skills pipeline depends on.

This skill runs once per project. All other skills assume these files exist.

---

## Your disposition

- **Idempotent.** Check before creating. If a file exists with correct content, skip it.
  If it exists with different content, warn the user and ask before overwriting.
- **Report everything.** Show a summary table of what was created, skipped, or updated.
- **No domain logic.** You set up infrastructure only. Types, runners, scripts, hooks.

---

## Step 1 — Audit existing infrastructure

Check which of these files already exist:

```
src/shared/spec-framework.ts
scripts/spec-tools.ts
scripts/generate-specs.ts
scripts/tsconfig.json
docs/specs.ts
.claude/hooks.json
```

Report status before proceeding:
> "Here's what I found:
> - `shared/spec-framework.ts` — missing (will create)
> - `scripts/spec-tools.ts` — exists (will check content)
> - ..."

---

## Step 2 — Create `shared/spec-framework.ts`

The single infrastructure file. Contains all types (`Result`, `SpecFn`, `Spec`,
`StepInfo`, `CanonicalFn`, example types, assertion types) AND the runners
(`testSpec`, `execCanonical`) AND utilities (`inheritFromSteps`, `asStepSpec`).

This replaces v3's two files (`spec.ts` + `testing.ts`) with one unified module.

See [examples.md](examples.md) for the complete file contents.

---

## Step 3 — Create `scripts/spec-tools.ts`

Pure functions for spec tree flattening and markdown generation:
- `flattenSpec` — walks the `steps` array recursively, collects failures in pipeline order
- `toMarkdownTable` — converts flat table to decision table markdown with symbols
- `toStepTable` — extracts pipeline step table from `StepInfo[]`

Adapts to v4's `steps: StepInfo[]` array (not v3's `Record<string, StepSpec>`).
Walks step specs via `step.spec.shouldFailWith` and recurses via `step.spec.steps`.

See [examples.md](examples.md) for the complete file contents.

---

## Step 4 — Create `scripts/generate-specs.ts`

Auto-discovers specs with `document: true` via glob — no manual manifest needed.
Globs for `src/**/*.spec.ts`, imports each file, and writes `.spec.md` files
next to any spec that has `document: true`. These are **fully generated** structural
docs — pipeline tables and decision tables. No prose sections, no manual editing.
The file is overwritten on every run.

Business-friendly prose documentation lives separately in `/docs/` and is managed
by the `ddd-documentation` skill.

See [examples.md](examples.md) for the complete file contents.

---

## Step 5 — Create `scripts/tsconfig.json`

Scripts live outside `src/` and need their own tsconfig. Targets ES2022, ESNext modules,
bundler resolution.

See [examples.md](examples.md) for the complete file contents.

---

## Step 6 — Create `docs/` directory structure

Create the `/docs` directory for Jekyll Just the Docs site. The `ddd-documentation`
skill populates this with business-friendly prose docs, but the structure is
established here:

```
docs/
  _config.yml              <- Jekyll Just the Docs configuration
  index.md                 <- Domain home page
```

See [examples.md](examples.md) for the complete file contents.

---

## Step 7 — Update `package.json`

Add the gen:specs script and glob + tsx dependencies:

```json
{
  "scripts": {
    "gen:specs": "tsx scripts/generate-specs.ts"
  },
  "devDependencies": {
    "glob": "^11.0.0",
    "tsx": "^4.0.0"
  }
}
```

Only add what's missing. Do not overwrite existing scripts or dependencies.

---

## Step 8 — Create `.claude/hooks.json`

Auto-regenerate `.spec.md` files when any `.spec.ts` file is written or edited.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "command": "bash -c 'if echo \"$TOOL_INPUT\" | grep -q \"\\.spec\\.ts\"; then npm run gen:specs 2>&1 | tail -5 || true; fi'"
      }
    ]
  }
}
```

If `.claude/hooks.json` already exists, **merge** — add the PostToolUse entry
without removing existing hooks.

---

## Step 9 — Install dependencies

Run `npm install` to install glob and tsx.

---

## Step 10 — Verify

Run `npm run gen:specs` to confirm the pipeline works. With no `document: true`
specs it should report "No specs with document: true found — nothing to generate."
and exit cleanly.

---

## Summary

After running, report:

```
ddd-init complete:

  shared/spec-framework.ts  created
  scripts/spec-tools.ts     created
  scripts/generate-specs.ts created
  scripts/tsconfig.json     created
  docs/_config.yml          created
  docs/index.md             created
  package.json              updated (gen:specs script, glob + tsx devDeps)
  .claude/hooks.json        created
  npm install               ran
  npm run gen:specs          verified

All infrastructure is ready. Use ddd-data-modelling or ddd-spec to start
building domain functions.
```

---

## Hard rules

- **Idempotent.** Never overwrite without asking. Check content, not just existence.
- **Never create domain files.** Types, specs, examples, implementations — those
  belong to the other skills.
- **Merge, don't replace** existing `package.json` and `hooks.json`.
- **All shared file content is defined here.** Other skills reference these files
  but do not create or modify them.
- **No manifest.** Specs opt in to `.spec.md` generation via `document: true`.
  The `generate-specs` script auto-discovers them via glob.
- **Report every action.** The user should see exactly what was created, skipped,
  or updated.

## Additional resources

- For project conventions, folder structure, and naming rules, see [reference.md](reference.md)
- For complete file contents for each step, see [examples.md](examples.md)
