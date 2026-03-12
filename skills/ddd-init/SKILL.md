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
scripts/spec-manifest.ts
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
`StepInfo`, example types, assertion types) AND the test runner (`testSpec`) AND
the `inheritFromSteps` utility.

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

## Step 4 — Create `scripts/spec-manifest.ts`

The manifest is the single registry of all composed specs (factories). One
`ManifestEntry` per factory with `name`, `specPath`, and `exportName`. Starts
empty — `ddd-spec` adds entries when it creates composed specs.

See [examples.md](examples.md) for the complete file contents.

---

## Step 5 — Create `scripts/generate-specs.ts`

Entry point that reads the manifest, flattens each spec, and writes `.spec.md` files
next to each `.spec.ts`. These are **fully generated** structural docs — pipeline
tables and decision tables. No prose sections, no manual editing. The file is
overwritten on every run.

Business-friendly prose documentation lives separately in `/docs/` and is managed
by the `ddd-documentation` skill.

**Manifest validation:** Before doing any work, the script validates every manifest
entry — checks the file exists and the named export is present. If any entry is
stale (renamed or deleted spec file), the script fails with a clear error pointing
to the broken entry. Fix the manifest and re-run.

See [examples.md](examples.md) for the complete file contents.

---

## Step 6 — Create `scripts/tsconfig.json`

Scripts live outside `src/` and need their own tsconfig. Targets ES2022, ESNext modules,
bundler resolution.

See [examples.md](examples.md) for the complete file contents.

---

## Step 7 — Create `docs/` directory structure

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

## Step 8 — Update `package.json`

Add the gen:specs script and tsx dependency:

```json
{
  "scripts": {
    "gen:specs": "tsx scripts/generate-specs.ts"
  },
  "devDependencies": {
    "tsx": "^4.0.0"
  }
}
```

Only add what's missing. Do not overwrite existing scripts or dependencies.

---

## Step 9 — Create `.claude/hooks.json`

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

## Step 10 — Install dependencies

Run `npm install` to install tsx.

---

## Step 11 — Verify

Run `npm run gen:specs` to confirm the pipeline works. With an empty manifest it
should report "spec-manifest.ts is empty — no specs to generate." and exit cleanly.

---

## Summary

After running, report:

```
ddd-init complete:

  shared/spec-framework.ts  created
  scripts/spec-tools.ts     created
  scripts/spec-manifest.ts  created
  scripts/generate-specs.ts created
  scripts/tsconfig.json     created
  docs/specs.ts             created
  package.json              updated (gen:specs script, tsx devDep)
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
- **The manifest starts empty.** The `ddd-spec` skill adds entries when it creates
  composed specs. This skill only creates the empty manifest file.
- **Report every action.** The user should see exactly what was created, skipped,
  or updated.

## Additional resources

- For project conventions, folder structure, and naming rules, see [reference.md](reference.md)
- For complete file contents for each step, see [examples.md](examples.md)
