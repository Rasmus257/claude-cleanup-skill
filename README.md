# claude-cleanup-skill

A Claude Code command that tidies the code on your current branch, and nothing else.

Four conservative passes, each a fresh subagent: dead code, simplification, extraction, comments. It stops after the report and waits for you to review the diff.

```
/cleanup
/cleanup the auth flow
/cleanup src/api src/lib/auth.ts
```

## Why

Review comments about an unused import, or about a comment that restates the line below it, waste a reviewer's time. So does a cleanup agent that decides your working code needs an interface, a factory, and three new files.

This one skews conservative. The bar for touching anything is "this materially helps the next reader", not "this is different". "I left it alone because X" is a valid outcome for an entire pass.

Two things pile up specifically in codebases that agents write.

The first is the same function, written five slightly different ways. Each session reached for a fresh implementation instead of finding the one already three directories over. Pass 2 hunts those and collapses them onto a single copy, but only when it can read every version and confirm they are actually the same. If one of them treats an empty list differently, they all stay and you get told about it. Copies that merely look alike are how a cleanup pass changes behavior while your typecheck stays green.

The second is comments written for the next agent. "NOTE for whoever picks this up", a paragraph re-explaining the function directly below it, a pointer to a similar helper elsewhere. The instinct is good and the result is a codebase where a third of what you load is narration. Pass 4 cuts the ones that narrate and keeps the ones carrying a constraint the code cannot show.

## Before and after

One file, after all four passes:

```diff
- import { readFileSync } from 'node:fs';
  import './telemetry.js';
  import { formatCurrency } from '../shared/money.js';

- // Formats cents as a dollar string.
- // NOTE for whoever picks this up: there is a similar helper in
- // shared/money.ts, but this one is local to the cart so they may drift.
- function formatPrice(cents: number): string {
-   return `$${(cents / 100).toFixed(2)}`;
- }
-
  export function cartTotal(items: Item[]): string {
-   // Sum the item prices
    let sum = 0;
    for (const item of items) sum += item.cents;
-   return formatPrice(sum);
+   return formatCurrency(sum);
  }
-
- function legacyCartLabel(items: Item[]): string {
-   return `${items.length} items`;
- }
```

Pass 1 dropped the unused `readFileSync` and the unreferenced `legacyCartLabel`, and left `import './telemetry.js'` alone because a bare side-effect import has zero references by design. Pass 2 read both `formatPrice` and the existing `formatCurrency`, confirmed they were identical, and pointed the call at the one the repo already exports. Pass 4 removed the comment explaining what the function below it did, and the note warning about the duplication that no longer exists.

Twenty-one lines to eight, and nothing behaves differently.

## Install

```sh
curl -o ~/.claude/commands/cleanup.md \
  https://raw.githubusercontent.com/Rasmus257/claude-cleanup-skill/main/cleanup.md
```

Then run `/cleanup` in any repo. The path is `commands/`, not `skills/`, because this is a slash command. Claude Code registers it as a skill either way.

To scope it to one project, put it in `.claude/commands/` there instead. Commit it and your team gets it too.

It deletes code, so read [`cleanup.md`](cleanup.md) before you run it. That one file is the whole tool.

## Scope

The branch diff is the outer bound. `/cleanup` resolves the base through `origin/HEAD`, then `origin/main`, `origin/master`, and finally local `main` or `master`. If none of them resolve it asks instead of guessing, because a failed lookup and a clean branch look identical otherwise.

Inside that bound, it follows what you asked for. Give it paths and it uses those. Describe what you want cleaned ("the checkout screen", "what we just changed") and it keeps only the matching files, then prints what it dropped so you can correct it. Say nothing and it takes the whole branch diff.

It never widens past the branch, and a request that matches nothing stops the run rather than falling back to cleaning everything.

Always carved out: lockfiles, generated and build output, vendored code, fixtures, templates, seeds, migrations, prose files, and anything marked `WIP` or `DO NOT COMMIT`.

## The four passes

Dead code runs first so later passes don't polish code that is about to be deleted. Comments run last so they describe the final shape.

| Pass | Removes | Leaves |
| --- | --- | --- |
| 1. Dead code | Unreferenced functions, imports, props, type fields; unreachable branches; vestigial naming left by a migration | Bare side-effect imports, anything reached reflectively or by name, public API surface, documented caller-parity params |
| 2. Simplification | Duplication a shared util already covers, hand-rolled stdlib, accidental O(n²), verbose guards | Three similar lines. No new abstractions, ever |
| 3. Extraction | Splits a file only when several cohesive groups share it, following the repo's existing layout | Everything else. The default position is "do not extract" |
| 4. Comments | Comments restating the code, rotting ticket references, pointers to deleted code, resolved TODOs | Every WHY comment, external-system quirks, tradeoff notes, magic-number citations. It adds no new comments |

Before deleting any symbol, a pass greps the whole repository for it, including string-based reachability, so DI containers, dynamic imports, route registration and serialization don't get quietly broken.

Between passes it runs your project's own typecheck or lint. Project-wide, not scoped to the touched files: a file-scoped `tsc` reports every path alias as a missing module, and it cannot see breakage in a file that imports what a pass moved. A broken tree ends the run. It also runs the check before pass 1, so a failure that was already there doesn't get blamed on a pass.

## What it won't do

- Add abstractions or comments. Pass 2 is barred from extracting a helper, and pass 4 may only edit comments that already exist.
- Fix bugs it finds. It reports them with `file:line` and moves on. A fix hiding in a cleanup diff is a fix nobody reviewed.
- Reformat. No `--write` pass over a file, because whole-file reformatting destroys the reviewability this exists to produce.
- Touch files outside the diff, or the carved-out ones.

## About "no behavior changes"

Every pass is told behavior must stay identical, and the counter-examples in the file exist to block the changes that look safe and aren't: deleting a side-effect import, turning a sequential loop parallel, dropping a branch that is only reached reflectively.

These are still model edits gated by your typecheck, not a proof. A typecheck cannot see a removed log line, a changed concurrency pattern, or a deleted import that existed purely for its side effect. Review the diff.

If your working tree is dirty when you start, it says so and stops. Nothing here commits, so uncommitted work it edits has no restore point. Commit or stash first.

## Requirements

Claude Code and a git repo.

## License

MIT
