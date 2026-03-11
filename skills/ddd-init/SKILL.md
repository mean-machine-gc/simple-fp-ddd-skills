---
name: ddd-init
description: >
  Initializes DDD project infrastructure: shared type files (spec.ts, testing.ts),
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
src/shared/spec.ts
src/shared/testing.ts
scripts/spec-tools.ts
scripts/spec-manifest.ts
scripts/generate-specs.ts
scripts/tsconfig.json
docs/specs.ts
.claude/hooks.json
```

Report status before proceeding:
> "Here's what I found:
> - `shared/spec.ts` — missing (will create)
> - `shared/testing.ts` — exists (will check content)
> - ..."

---

## Step 2 — Create `shared/spec.ts`

Core type definitions used by all other files: `Result<T, F, S>`, spec predicates
(`ConstraintPredicate`, `ConditionPredicate`, `AssertionPredicate`), `FunctionSpec`
(constraints with predicates + inline examples), `FactorySpec` (steps + inline failure
examples), `ShellFactorySpec` (factory + baseDeps + dep propagation), `StepSpec`
(union of both — used in `FactorySpec.steps` to allow nesting), and example types
(`FailureExample`, `SuccessExample`, `MixedFailureExample`, `OneOrMore`).

See [examples.md](examples.md) for the complete file contents.

---

## Step 3 — Create `shared/testing.ts`

Test runners that read predicates and examples directly from the spec object.
Exports `runSpec` (sync atomic), `runSpecAsync` (async atomic), `runFactorySpec`
(sync factory), `runFactorySpecAsync` (async factory), and `runShellSpec`
(async factory + dep propagation).

For each success example, runners verify up to four things:
1. Condition predicate returns `true`
2. Spec assertions pass (atomic functions only — factories skip this)
3. `result.value` matches the expected `then` value
4. `result.successType` contains the expected success type key

See [examples.md](examples.md) for the complete file contents.

---

## Step 4 — Create `scripts/spec-tools.ts`

Pure functions for spec tree flattening and markdown generation:
- `flattenSpec` — recursively walks factory spec tree, collects constraints in pipeline order
- `toMarkdownTable` — converts flat table to decision table markdown with ✓/✗/— symbols
- `toStepTable` — extracts pipeline step table for §4/§5 of `.spec.md`

See [examples.md](examples.md) for the complete file contents.

---

## Step 5 — Create `scripts/spec-manifest.ts`

The manifest is the single registry of all factory specs. One `ManifestEntry` per
factory with `name`, `specPath`, and `exportName`. Starts empty — `ddd-spec` adds
entries when it creates factory specs.

See [examples.md](examples.md) for the complete file contents.

---

## Step 6 — Create `scripts/generate-specs.ts`

Entry point that reads the manifest, flattens each spec, and writes `.spec.md` files.
Only regenerates content between `<!-- BEGIN:GENERATED -->` / `<!-- END:GENERATED -->`
markers. Prose sections (§1-§3) written by `ddd-documentation` are preserved.

**Manifest validation:** Before doing any work, the script validates every manifest
entry — checks the file exists and the named export is present. If any entry is
stale (renamed or deleted spec file), the script fails with a clear error pointing
to the broken entry. Fix the manifest and re-run.

**Staleness detection:** When generated content actually changes, the script injects
a `<!-- ⚠ STALE: ... -->` marker above each prose section, signaling that the
business scenarios may need updating. `ddd-documentation` removes these markers
when it updates prose.

See [examples.md](examples.md) for the complete file contents.

---

## Step 7 — Create `scripts/tsconfig.json`

Scripts live outside `src/` and need their own tsconfig. Targets ES2022, ESNext modules,
bundler resolution.

See [examples.md](examples.md) for the complete file contents.

---

## Step 8 — Create `docs/specs.ts`

Documentation registry listing all generated `.spec.md` files. Updated automatically
by `npm run gen:specs`. Developers import this for a single access point to all spec docs.

See [examples.md](examples.md) for the complete file contents.

---

## Step 9 — Update `package.json`

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

## Step 10 — Create `.claude/hooks.json`

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

## Step 11 — Install dependencies

Run `npm install` to install tsx.

---

## Step 12 — Verify

Run `npm run gen:specs` to confirm the pipeline works. With an empty manifest it
should report "spec-manifest.ts is empty — no specs to generate." and exit cleanly.

---

## Summary

After running, report:

```
ddd-init complete:

  shared/spec.ts            ✓ created
  shared/testing.ts         ✓ created
  scripts/spec-tools.ts     ✓ created
  scripts/spec-manifest.ts  ✓ created
  scripts/generate-specs.ts ✓ created
  scripts/tsconfig.json     ✓ created
  docs/specs.ts             ✓ created
  package.json              ✓ updated (gen:specs script, tsx devDep)
  .claude/hooks.json        ✓ created
  npm install               ✓ ran
  npm run gen:specs          ✓ verified

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
  factory specs. This skill only creates the empty manifest file.
- **Report every action.** The user should see exactly what was created, skipped,
  or updated.

## Additional resources

- For project conventions, folder structure, and naming rules, see [reference.md](reference.md)
- For complete file contents for each step, see [examples.md](examples.md)
