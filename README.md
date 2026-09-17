# cleanup

A Claude Code command that tidies the diff on your current branch — and only the diff on your current branch.

Four conservative passes, each a fresh subagent: **dead code**, **simplification**, **extraction**, **comments**. It stops after the report and waits for you to review. It never commits, and it never changes behavior.

```
/cleanup
```

## Why

Review comments about an unused import or a comment that restates the line below it are a waste of a reviewer. So is a cleanup agent that decides your working code needs an interface, a factory, and three new files.

This one is built to skew conservative. The bar for touching anything is "this materially helps the next reader", not "this is different". "I left it alone because X" is a valid outcome for a whole pass.

## Install

```
/plugin marketplace add Rasmus257/cleanup
/plugin install cleanup@cleanup
```

Then `/cleanup` in any repo. Nothing else to configure — the subagent it uses ships with the plugin.

## The four passes

Passes run in order, not in parallel: dead code first so later passes don't polish code that's about to be deleted, comments last so they describe the final shape. Between each pass it runs your project's own typecheck or lint against the touched files, and stops the run if the tree broke.

| Pass | Removes | Leaves |
| --- | --- | --- |
| **1. Dead code** | Unreferenced functions, imports, props, type fields; unreachable branches; resolved TODOs; comments describing deleted code | Anything reached reflectively or by name, public API surface, documented caller-parity params |
| **2. Simplification** | Inline duplication where a shared util already exists, hand-rolled stdlib, accidental O(n²), verbose guards | Three similar lines. No new abstractions, ever |
| **3. Extraction** | Splits a file only when several cohesive groups are sharing it, following the repo's existing conventions | Everything else. The default position is "do not extract" |
| **4. Comments** | Comments restating the code, rotting ticket references, pointers to deleted code, stale TODOs | Every WHY comment, external-system quirks, tradeoff notes, magic-number citations. It adds no new comments |

## Scope

`/cleanup` works on `git diff <default-branch>...HEAD` plus your uncommitted changes. Pass paths to scope it yourself:

```
/cleanup src/api src/lib/auth.ts
```

Carved out either way: lockfiles, generated and build output, vendored code, fixtures, templates and seeds, migrations, and anything marked `WIP` or `DO NOT COMMIT`.

## What it won't do

- Change behavior. Every code path, response shape, error message, log line and effect order stays identical.
- Fix bugs it finds. It reports them with `file:line` and moves on — a fix hiding in a cleanup diff is a fix nobody reviewed.
- Add dependencies, abstractions, or comments.
- Commit, push, or open anything. You review the diff.
- Keep going on a broken tree.

## Requirements

Claude Code with plugin support, and a git repo with a remote. That's it.

## License

MIT
