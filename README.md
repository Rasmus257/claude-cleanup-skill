# claude-cleanup-skill

A Claude Code command that tidies the diff on your current branch — and only the diff on your current branch.

Four conservative passes, each a fresh subagent: **dead code**, **simplification**, **extraction**, **comments**. It stops after the report and waits for you to review the diff.

```
/cleanup
```

## Why

Review comments about an unused import, or about a comment that restates the line below it, are a waste of a reviewer. So is a cleanup agent that decides your working code needs an interface, a factory, and three new files.

This one is built to skew conservative. The bar for touching anything is "this materially helps the next reader", not "this is different". "I left it alone because X" is a valid outcome for an entire pass.

## Install

One file, no plugin, no dependencies:

```sh
curl -o ~/.claude/commands/cleanup.md \
  https://raw.githubusercontent.com/Rasmus257/claude-cleanup-skill/main/cleanup.md
```

Then `/cleanup` in any repo. Note the path is `commands/`, not `skills/` — this is a slash command, which Claude Code registers as a skill. Drop it in `.claude/commands/` inside a project instead if you'd rather scope it to one repo, or commit it there to share it with your team.

It deletes code, so read [`cleanup.md`](cleanup.md) before you run it. That's the whole tool — one file, one screen at a time.

## The four passes

Passes run in order, not in parallel: dead code first so later passes don't polish code that's about to be deleted, comments last so they describe the final shape. Between each pass it runs your project's own typecheck or lint — project-wide, not scoped to the touched files, because a file-scoped `tsc` reports every path alias as a missing module and can't see breakage in a file that imports what a pass moved. It stops the run if the tree broke.

| Pass | Removes | Leaves |
| --- | --- | --- |
| **1. Dead code** | Unreferenced functions, imports, props, type fields; unreachable branches; vestigial naming left by a migration | Bare side-effect imports, anything reached reflectively or by name, public API surface, documented caller-parity params |
| **2. Simplification** | Inline duplication where a shared util already exists, hand-rolled stdlib, accidental O(n²), verbose guards | Three similar lines. No new abstractions, ever |
| **3. Extraction** | Splits a file only when several cohesive groups are sharing it, following the repo's existing conventions | Everything else. The default position is "do not extract" |
| **4. Comments** | Comments restating the code, rotting ticket references, pointers to deleted code, stale TODOs | Every WHY comment, external-system quirks, tradeoff notes, magic-number citations. It adds no new comments |

Before deleting any symbol, a pass greps the whole repository for it — including string-based reachability, so DI containers, dynamic imports, route registration and serialization don't get quietly broken.

## Scope

`/cleanup` works on `git diff origin/<default-branch>...HEAD`. It resolves the base through `origin/HEAD` → `origin/main` → `origin/master` → local `main`/`master`, and asks rather than guessing if none resolve — a failed lookup is never treated as an empty diff.

Pass paths to scope it yourself:

```
/cleanup src/api src/lib/auth.ts
```

Carved out either way: lockfiles, generated and build output, vendored code, fixtures, templates and seeds, migrations, prose (`*.md`, `*.txt`) unless you pass them explicitly, and anything marked `WIP` or `DO NOT COMMIT`.

## What it won't do

- Add abstractions, or add comments. Pass 2 is explicitly barred from extracting a helper, and pass 4 may only edit comments that already exist.
- Fix bugs it finds. It reports them with `file:line` and moves on — a fix hiding in a cleanup diff is a fix nobody reviewed.
- Reformat. No `--write` pass over a file; whole-file reformatting destroys the reviewability this exists to produce.
- Touch files outside the diff, or the carved-out ones.
- Keep going on a broken tree. It runs your check before pass 1 too, so a pre-existing failure doesn't get blamed on a pass.

## About "no behavior changes"

Every pass is told that behavior must stay identical, and the ❌ examples exist to block the changes that look safe and aren't — deleting a side-effect import, turning a sequential loop parallel, dropping a branch that's only reached reflectively.

But these are model edits gated by your typecheck, not a proof. A typecheck cannot see a removed log line, a changed concurrency pattern, or a deleted import that existed purely for its side effect. Treat it as a tool that skews hard toward leaving things alone, and review the diff.

If your working tree is dirty when you start, it says so and stops. Nothing here commits, so uncommitted work it edits has no restore point — commit or stash first.

## Requirements

Claude Code and a git repo. If the default branch can't be resolved it asks rather than guessing.

## License

MIT
