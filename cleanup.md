---
description: Multi-pass cleanup of the whole current-branch diff (dead code, simplification, extraction, comments) via four sequential subagents. Only when the user explicitly asks to clean up or tidy a branch or PR, not to tidy a single file they're editing.
argument-hint: "[paths, or what you want cleaned up]"
disable-model-invocation: false
---

Tidy the code introduced on the current branch through four sequential subagent passes: **dead code**, **simplification**, **extraction**, **comments**.

The goal is a tidier diff, not a rewrite. You are the orchestrator: resolve scope, dispatch the passes, run the checks between them, write the report. You don't edit code yourself.

## 1. Checkpoint

If `git status --porcelain` is non-empty, say so and stop until the user commits, stashes a copy (`git stash push -u -m pre-cleanup`), or explicitly accepts that uncommitted work gets edited with no way back. The passes rewrite files in place and nothing here commits, so un-committed work has no restore point. Don't skip this because the changes look small.

## 2. Resolve scope

1. **Base ref.** First that resolves under `git rev-parse --verify --quiet`: `origin/HEAD`, `origin/main`, `origin/master`, then local `main`, `master`. Keep the `origin/` prefix. Branching off a freshly fetched `origin/main` while local `main` lags behind makes `main...HEAD` include the commits in between: other people's merged work, fed to four deletion passes.
2. **Changed files**: `git diff --name-only --diff-filter=d <base>...HEAD`, plus `git status --porcelain -uall`. From the status output, strip the two-character prefix; for a rename (`R  old -> new`) take the path after the arrow; unquote paths git wrapped in `"`. `--diff-filter=d` drops deleted files so no pass is told to edit one, and `-uall` lists untracked files individually instead of naming their directory.
3. **A failed resolution is not an empty diff.** `git symbolic-ref … | sed` exits 0 with an empty string when the ref is missing, and `git diff ...HEAD` then accepts it and returns nothing, which is indistinguishable from a clean branch. If no base ref resolves, or the diff fails with `no merge base` or `ambiguous argument`, stop and ask for the base branch.
4. **Narrow to what was actually asked for.** `$ARGUMENTS` holds the user's request. It is empty, a list of paths, or a description of what they want cleaned ("the auth flow", "what we just changed in checkout", "the deploy screen"). Paths replace the diff. A description narrows it: keep only the changed files that request covers, then print which files you dropped and why, so the user can correct you before anything is dispatched. Empty means the whole branch diff.

   The branch is the outer bound in every case. Never widen past it, and never let a description pull in a file the diff doesn't contain. If the request matches nothing in the diff, say so and stop rather than falling back to cleaning everything.
5. **Carve out**, even when the diff touches them:
   - Lockfiles, and `requirements.txt` when it's a lock output rather than hand-curated
   - Generated and build output: `dist/`, `build/`, `out/`, `target/`, `.next/`, `node_modules/`, `vendor/`, `__pycache__/`, and files matching `*.generated.*`, `*_pb2.py`, `*.pb.go`
   - Fixtures, templates, seeds, snapshots, `testdata/`
   - Vendored code and third-party type stubs the repo doesn't author
   - Migrations, an append-only historical record rather than code to tidy
   - Prose: `*.md`, `*.mdx`, `*.txt`, unless the user passed them as arguments
   - Anything marked `WIP`, `LOCAL-ONLY`, or `DO NOT COMMIT`
6. Empty list → print "nothing on this branch to clean" and stop.
7. Print the scope. Over 25 files, summarize by area and confirm first. Over 150, say it's too large to be worth four passes and ask for paths.

## 3. Establish the check

Find the project's own check, in order: a `package.json` script named `typecheck`, `tsc`, `check`, or `lint`; a `Makefile` target of the same name; `pyproject.toml` (`ruff`, `mypy`, `ty`); `cargo check`; `go build ./...`. Use the lockfile only to pick the runner (`package-lock.json` → npm, `pnpm-lock.yaml` → pnpm, `bun.lock` → bun). If the repo has no check, say so and run the passes ungated. Don't invent one.

Run it **before pass 1**. If the tree is already broken, say so and ask whether to continue, so no pass gets blamed for a pre-existing failure.

Re-run **the same project-wide command** after each pass. Don't narrow it to the touched files: `tsc` ignores `tsconfig.json` when handed file paths, so a per-file run reports every path alias as a missing module, and it can't see breakage in an unlisted file that imports what a pass moved. Lint may be scoped to the touched files; the typecheck may not.

A pass that breaks the tree ends the run. Report and stop, don't dispatch the next one.

## 4. Run the four passes

One fresh `general-purpose` subagent per pass, sequential: dead code first so later passes don't polish code that's about to be deleted, comments last so they describe the final shape. Track them as todos.

Each dispatch gets the **standard brief** verbatim, then the exact in-scope file list, then that pass's section. Don't make the subagent rediscover scope or read this file.

**After pass 3, recompute the file list** (same commands, same carve-outs) before dispatching pass 4, so files pass 3 created get the comment pass and land in the report.

### The standard brief

> Run exactly one cleanup pass over the file list below. Do that pass and nothing else. You are not improving the code. You are making the diff easier to read, at zero behavioral risk.
>
> These files are the diff of branch `<branch>` against `<base>`. To see what this branch changed in one, run `git diff <base>...HEAD -- <file>`.
>
> **Behavior stays identical.** Every code path, return value, response shape, error message, log line, raised exception, concurrency pattern and side-effect order is what it was before you touched the file. If you can't convince yourself a change is behavior-preserving, it isn't. Leave it.
>
> **Edit only the listed files.** Read anything in the repo you need to, since proving something is unreferenced requires it, but confine edits to the list. The one exception is the extraction pass, which may create files inside the repo's existing layout.
>
> **Stay in your pass.** Spotting a great simplification during the dead-code pass means leaving it for the pass that owns it.
>
> **Before deleting any symbol, grep the whole repository for it**, not just the changed files and not just exact call syntax. A symbol reached only by a string is still reached: dependency injection, serialization, dynamic import, route and CLI registration, framework naming conventions, template references, reflection. When in doubt, it stays.
>
> **Don't run the repo's formatter** or any `--write` pass over a file. Reformatting whole files destroys the reviewability this exists to produce.
>
> **Report bugs, don't fix them.** A fix inside a cleanup pass is a behavior change hiding in a diff nobody is reviewing for correctness. Note the `file:line` and move on.
>
> Conservative wins every tie. "I left this alone because X" is a good outcome; a confident wrong deletion is the failure mode.
>
> Report back: files edited with LOC deltas, the highest-value changes as `file:line` plus one line each on why, what you left alone and why, and bugs noticed but not fixed. Brief, because four of these get aggregated into one report.

### Pass 1: Dead code

Remove what's verifiably unreferenced: unused functions, classes, exports, imports, props, parameters, type fields; unreachable branches and constant-false guards; vestigial naming left behind by a migration. Delete a comment only when it sits directly on code you're deleting. All other comment work belongs to pass 4.

- ❌ Deleting a bare side-effect import (`import './globals.css'`, `import 'reflect-metadata'`, `import './registerHandlers.js'`). It has zero references because that is what it is for, and removing it typechecks clean.
- ❌ Deleting an exported handler because nothing imports it. It is registered by name in a route table.
- ❌ Deleting an unused parameter on one implementation of an interface.
- ✅ Deleting a *named* import whose binding has zero references in the file.
- ✅ Deleting a private helper whose only caller was removed earlier in this same branch.

Leave: public API surface, documented caller-parity params, anything imported by files not on the list.

### Pass 2: Simplification

Replace with what already exists: duplication a shared util in this repo already covers, boilerplate the language or framework provides, accidental O(n²) where O(n) is one line away, verbose guards a single optional chain or default covers, type narrowing the control flow already implies.

- ❌ Extracting a new helper because three call sites share five lines.
- ❌ Collapsing an if/else chain into nested ternaries. Shorter, harder to read.
- ❌ Replacing a sequential await loop with a parallel primitive. Concurrency is a behavior change: rate limits, pool pressure, and what has already happened when one item throws all differ. Report it as a possible speedup instead.
- ✅ Replacing a hand-rolled dedup loop with the language's set type.
- ✅ Replacing a nested-loop membership test with a set lookup.

Hard rule: **no new abstractions.** Extraction earns its keep only when the duplication is a genuine hotspot, the right abstraction is obvious, and every call site benefits. Three similar lines beat a premature abstraction.

### Pass 3: Extraction

Split a file only when several cohesive groups are sharing it and each would read better alone. Follow the repo's existing layout, and read sibling directories before inventing a structure.

**Move a symbol only when every reference to it is inside the listed files**, so grep first. If anything outside the list imports it, leave it, or re-export from the original path so no unlisted file has to change.

- ❌ Splitting a 400-line file into six because 400 is a big number.
- ❌ Creating a one-file package, or a namespace with a single module in it.
- ❌ Moving a helper away from its only caller.
- ✅ Moving three self-contained components out of a page file into the components directory the repo already uses.

Default position: **do not extract.** The bar is "the next reader holds this file in their head better as a result".

### Pass 4: Comments

Remove comments restating what the code says, ticket and PR references that rot, pointers to code that no longer exists, resolved TODOs, and doc comments echoed again in the body.

- ❌ Removing `// 5000ms, from the observed p99`. That is a cited magic number.
- ❌ Removing `// the API returns 200 with an error body here`. That is an external contract.
- ❌ Paraphrasing a subtle WHY into something shorter but vaguer.
- ✅ Removing `// loop through the items` above a loop over items.
- ✅ Removing `// Added for ABC-123`.
- ✅ Tightening six lines of history into two that keep the reason.

Keep untouched: non-obvious WHY comments (constraints, invariants, bug workarounds, justifications for something that looks wrong at first glance), external-system quirks, tradeoff notes that live nowhere else, and citations for magic numbers.

Hard rule: **do not add new comments.** Only edit existing ones.

## 5. Report

One consolidated report after the four passes, or after the one that aborted:

- Files touched with LOC delta each, and the total
- The highest-value changes, `file:line` plus a one-line why
- What was deliberately left alone, and why it survived
- Bugs and smells noticed but not fixed: `file:line`, what's wrong, why it was out of scope
- Final check status
- How to undo: `git diff` to review, `git checkout -- <file>` to drop a file's changes

Then stop, and let the user review the diff.
