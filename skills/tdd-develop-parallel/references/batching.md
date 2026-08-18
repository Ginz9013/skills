# Batching, quarantine, and commits

## Frontier

The frontier is every ticket whose blockers are all `done` and whose own status is not `done` or `in-progress`.

If the frontier is empty and nothing is in flight, the board is finished. If the frontier is empty but tickets remain, the board is deadlocked — report the cycle rather than picking a ticket anyway.

## Blast radius

A frontier is permission to start, not permission to parallelise. Two tickets that edit the same file will silently overwrite each other in a shared working tree, and the loser's work is gone without a conflict marker to warn you.

Each ticket carries its write ownership from the board. Verify it rather than trusting it: read the ticket, then grep for the symbols and paths it names. When the board's list looks wrong, fix it on the board before dispatching — that list is load-bearing twice over.

Then partition:

- **Batch** — frontier tickets with pairwise disjoint write ownership.
- **Queue** — everything else, held for a later round.

<hard-rules>
- A ticket marked `unbounded` runs alone. So does one whose ownership you could not establish.
- A wide mechanical change — a rename, a shared type change — is never batched with anything.
- Two tickets needing the same shared resource are not batchable unless the board allocates that resource per worker.
- Cap the batch at 4. Beyond that, integration review costs more than the parallelism saves.
</hard-rules>

If batch members genuinely must touch overlapping files and the user still wants them parallel, dispatch with worktree isolation and say so out loud — you are trading collision risk for merge cost, and the merge is yours.

## Quarantine

A per-ticket commit only means something if the tree contains nothing but work that passed. Stash each non-committable ticket's files, tagged by ticket:

```bash
git stash push -m "wip/<feature-slug>/<NN>" -- <that ticket's files>
```

Record every stash ref in the report; it is the only handle on that work. Because batch members own disjoint files by construction, these stashes pop back cleanly once the commits have landed.

If quarantining empties the tree, the round produced nothing. Report why and recompute the frontier.

## Verify before committing

On the quarantined tree, before any commit:

- Run the **full** suite, not the subset each worker ran.
- Run the static checks, comparing against the board's baseline rather than expecting zero.

Green here is the claim each commit is about to make, so make it true first. Fix small integration breakage yourself now; anything larger becomes a new ticket, and that ticket's files get quarantined too.

Record the current `HEAD` sha. That is the round's **base**, and the review runs against it.

## Commit

In dependency order, one commit per ticket, staged by that ticket's file list — never `git add -A`, never `git commit -a`.

```bash
git add <that ticket's files>
git commit
```

Match the repository's existing convention: read `git log` first and match its language, not just its format. Add a trailer so the board stays greppable after the working directory is gone:

```
<type>(<scope>): <subject>

<why this change, when the ticket title does not already say it>

Ticket: <feature-slug>/<NN>
```

After each commit, check that `git status` still shows the remaining tickets' files untouched. If a commit swept up a file you did not list, stop — the ownership map was wrong, and every commit after it is suspect.

Integration fixes belonging to no single ticket are their own commit, last.

## Review after committing

Run the two-axis review against the recorded base sha, **after** committing rather than before. A review tool that reads `git diff <fixed-point>...HEAD` cannot see staged or working-tree changes; pointing it at an uncommitted batch reviews nothing and reports clean. The failure is quiet, which is what makes it dangerous.

Findings become fix-up commits when small, or new tickets when not. Do not amend the ticket commits — the point of one commit per ticket is that it records what that ticket actually produced, review findings included.

## Closing the round

- Set the status of each landed ticket. The next frontier is computed from these, so a round that lands code without closing tickets freezes the board.
- `partial` tickets: restore their stashed work so a worker can resume.
- `blocked` tickets: leave the work stashed and restart them clean when the blocker clears.
- `unverified` tickets: leave them quarantined until the user decides.
- Report a table of ticket, commit sha, and status. A prose summary is not a substitute.
