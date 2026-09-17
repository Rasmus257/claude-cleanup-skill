---
description: Clean up the code on the current branch — dead code, simplification, extraction, comment hygiene — without behavior changes
argument-hint: "[optional paths to scope to]"
disable-model-invocation: false
---

Clean up the code introduced on the current branch through four sequential subagent passes: **dead code**, **simplification**, **structure / extraction**, **comments**.

The goal is a tidier diff. Not a rewrite, not a refactor of working code. Skew conservative across the whole flow — the bar to remove or restructure anything is "this materially helps the next reader", not "this is different".

## 1. Resolve scope

1. Find the default branch: `git symbolic-ref refs/remotes/origin/HEAD --short | sed 's@^origin/@@'`. Fall back to `main`, then `master`, if that fails.
2. Get the in-branch changed-files list: `git diff --name-only <default>...HEAD`. Add anything uncommitted with `git status --porcelain`.
3. If the user passed paths as arguments, use those as the scope instead of the diff. The carve-outs below still apply.
4. **Carve out these files even if the diff touches them:**
   - Lockfiles: `pnpm-lock.yaml`, `package-lock.json`, `yarn.lock`, `bun.lock`, `uv.lock`, `poetry.lock`, `Cargo.lock`, `go.sum`, `Gemfile.lock`, `composer.lock`, and `requirements.txt` when it's a lock output rather than hand-curated
   - Generated / build output: anything under `dist/`, `build/`, `out/`, `target/`, `.next/`, `node_modules/`, `vendor/`, `__pycache__/`, `.turbo/`, and files matching `*.generated.*`, `*_pb2.py`, `*.pb.go`
   - Template / scaffold / seed / fixture directories: `template/`, `templates/`, `fixtures/`, `seeds/`, `testdata/`, `__snapshots__/`
   - Vendored code and third-party type stubs the repo doesn't author
   - Migration files — they are an append-only historical record, not code to tidy
   - Anything tagged `LOCAL-ONLY`, `DO NOT COMMIT`, or `WIP` in a comment
   - Untracked scratch work: plans, specs, and notes the user deliberately keeps out of git
5. If the in-scope list is empty, print "nothing on this branch to clean" and stop.
6. Print the scope list. If it's over 25 files, summarize by area ("32 files: web/marketing, api/utils, …") and ask for confirmation before dispatching subagents.

## 2. Run four sequential cleanup passes

Each pass is one fresh `cleanup:cleanup-pass` subagent. Sequential, not parallel: dead code first so later passes don't waste effort on it, comments last so they describe the final shape.

**Between passes**, run a compile/lint check scoped to the touched files. Derive the command from the repo rather than assuming — check `package.json` scripts, the lockfile for the package manager, `pyproject.toml`, `Makefile`, `Cargo.toml`, or the CI workflow. Typical shapes:

| Stack | Check |
| --- | --- |
| TypeScript | `tsc --noEmit` from the owning workspace |
| JS / TS lint | the repo's own `lint` script |
| Python | `ruff check <files>` or the configured linter |
| Go | `go build ./...` |
| Rust | `cargo check` |

If a pass left the tree broken, surface it and stop — do **not** run later passes on a broken tree.

### Pass 1 — Dead code

Brief the subagent: find and remove genuinely dead code in the in-scope files. The bar is "verified unreferenced", not "looks unused". Grep the whole repo before deleting, not just the diff.

Target list:
- Unused functions, classes, methods, exports
- Unused imports, props, parameters, type fields
- Unreachable branches, useless guards, `if False:` / `if (false)` blocks
- Stale TODOs and FIXMEs pointing at work that's done
- Comment blocks describing code that has since been deleted
- Vestigial references to removed flows (e.g. lingering "snapshot" naming after a snapshot-to-git migration)

Leave alone: documented "kept for caller parity" params, anything reached reflectively or by name (DI containers, serialization, dynamic imports, framework conventions), anything a public API contract exposes, and anything imported by out-of-scope files.

### Pass 2 — Simplification

Brief the subagent: find inline duplication and obvious efficiency wins.

Target list:
- Logic copy-pasted across files where a shared util **already exists** — use it, don't create new ones
- Boilerplate the language or framework already provides (manual array dedup → a set, hand-rolled parallel awaits → the stdlib primitive)
- O(n²) loops where O(n) is one line away
- Verbose null/undefined guards a single optional chain or default would cover
- Type narrowing already implied by control flow

Hard rule: **no new abstractions**. If three call sites repeat the same five lines, leaving them is fine. Extraction only earns its keep when the duplication is a genuine hotspot, the right abstraction is obvious, and every call site benefits. Three similar lines beat a premature abstraction.

### Pass 3 — Component / module extraction

Brief the subagent: split oversized files into focused units **only when it materially helps readability**, never for LOC reduction.

Target list:
- Single files holding several cohesive groups of components or helpers that would each read better on their own
- Nothing else — helpers that are clearly private to one caller stay next to that caller

Match the conventions already in the repo. Read the sibling directories before inventing a structure: if the project colocates components under a `components/` or `_components/` folder, follow that; if modules live beside the router or runtime they serve, follow that. Don't create a one-file package or an empty namespace.

Default position: **do not extract**. The bar is "the next reader can hold this file in their head better as a result". Splitting for the sake of splitting is the failure mode here.

### Pass 4 — Comment cleanup

Brief the subagent: trim what-comments and rotting context, keep the why-comments, tighten the verbose ones.

**Remove:**
- Comments restating what well-named code already says ("initialize state", "loop through items", "helper function for X")
- Ticket, PR and fix references that rot ("Added for ABC-123", "Previously this did X but we changed it…")
- Pointers to code that no longer exists
- TODOs whose work is done
- Doc comments echoed again inside the function body

**Keep — don't even touch:**
- Non-obvious WHY comments: constraints, invariants, workarounds for a specific bug, justifications for something that looks wrong at first glance
- External-system contracts: third-party API quirks, framework dev-mode oddities, webhook payload shapes, auth session edge cases
- Tradeoff and decision notes that live nowhere else
- Inline citations for magic numbers and heuristics ("5000ms, from the observed p99")

**Tighten in place** when a six-line block becomes two clear lines with the WHY intact. If tightening means paraphrasing away nuance, leave it.

Hard rule: **do not add new comments.** Only edit existing ones.

## 3. Hard constraints across all four passes

- **No behavioral changes.** Every code path, response shape, error message, log call and effect order stays identical.
- **No "while I'm here" fixes.** A real bug gets reported, not fixed.
- **No file moves** outside the per-feature module patterns the codebase already uses.
- **No new dependencies.**
- **Never auto-commit.** The user reviews the consolidated diff and commits.
- **Stop at the first sign of breakage.** A broken tree mid-cleanup is worse than no cleanup.

## 4. Final report

After the four passes — or the earliest one that aborted — present one consolidated report:

- Files touched with the LOC delta per file, and the total
- The five or so highest-value removals and simplifications, each with `file:line` and a one-line why
- What was deliberately left alone, and why it survived
- Bugs and smells noticed but not fixed: `file:line`, what's wrong, why it was out of scope
- Final lint / typecheck status

Then stop and wait for the user to review the diff.

## Notes for the orchestrator

- Track the four passes as todos so the user can see where the run is.
- Give each subagent the **exact scope file list** so it doesn't re-discover scope.
- Brief each subagent on that pass's criteria from this file. Don't make it read the whole file.
- Run every pass even if the previous one found nothing — the criteria don't overlap.
- Tell each subagent which check command you verified for this repo, so it can sanity-check its own edits.
