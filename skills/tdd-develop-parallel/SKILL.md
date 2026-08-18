---
name: tdd-develop-parallel
description: Run the complete interface-first TDD workflow for one change with parallel workers — plan a board, then dispatch its tickets in collision-safe batches to sub-agents, landing one commit per ticket, then review. Use only when the user explicitly invokes this skill or asks to run their TDD workflow with parallel sub-agents. For a single-process run, use tdd-develop instead.
---

# TDD Develop (parallel)

You are the integrator. You never implement a ticket yourself, you are the only one who touches git, and you never trust a worker's report as verification.

Work proceeds one **round** at a time: compute the frontier, cut it down to tickets that cannot collide, dispatch those in parallel, then land one commit per ticket that passed.

## 0. Preconditions

Check these before planning. Each failure is a reason to degrade to `tdd-develop`, not to stop.

- **Git.** The quarantine and per-ticket commit model requires it. Without git, plan here but execute sequentially.
- **Sub-agent support.** The platform must be able to run workers concurrently. Without it, the whole board is sequential.
- **Test isolation.** Concurrent runs must not fight over ports, database schemas, or temporary directories. When they would, either allocate per worker in the board or run sequentially.
- **One session per repository.** `refs/stash` is shared across the whole repository, worktrees included. A second dispatcher stashing concurrently can make quarantined work vanish from under this one.

Say which preconditions failed and what you are doing about it. Silent degradation is worse than serial execution.

## 1. Plan

Use the installed `tdd-plan` skill to produce the board. Tell it:

- the change being planned;
- **execution mode: parallel** — the board must be written to files, because workers cannot see this conversation;
- the publication target — a tracker, or none.

If `tdd-plan` is not installed, say so and stop rather than improvising. Get the board approved before any code is written.

## 2. Each round

Read [references/batching.md](references/batching.md) for the frontier, blast-radius, quarantine, and commit rules. Per round:

1. **Frontier** — every ticket whose blockers are done and whose own status is not done or in progress. Empty frontier with nothing in flight means the board is finished.
2. **Batch** — cut the frontier down to tickets with pairwise disjoint write ownership. Everything else queues for a later round.
3. **Confirm** — show the user the frontier, the batch, the queue, and *why* each queued ticket was held back: which ticket it overlaps, on which files. Get approval. This is where the user catches the overlap you missed; do not skip it to save a round trip.
4. **Dispatch** — spawn one worker per batch ticket, all in a single message, so they actually run concurrently. Read [references/delegation-packet.md](references/delegation-packet.md) for what each worker gets.
5. **Triage** — read every report before touching git.
6. **Quarantine** — stash everything not committable this round, tagged by ticket.
7. **Verify** — on the quarantined tree, run the full suite and the static checks yourself, before any commit. Parallel work passes in isolation and fails together. Record `HEAD` as the round's base.
8. **Commit** — one commit per ticket, in dependency order, staged by that ticket's file list. Never `git add -A`.
9. **Review** — run the two-axis review against the recorded base, after committing.
10. **Close** — set each landed ticket's status, restore `partial` work, report a ticket-to-commit table, then return to step 1 and report the new frontier. Do not run the whole board unattended.

## 3. Judging a report

A worker's report is a claim, not a result. Sort the batch:

- **done** — candidate for commit, *after* its evidence passes the rules in `tdd-implement`'s execution report reference. Count red-green pairs against acceptance criteria; check that expected and observed failures match; in worktree mode, check the per-slice commits exist.
- **unverified** — the acceptance criteria may be met but the red-green claim is unsupported. Quarantine it and put the decision to the user: redo the ticket, or accept it knowingly. Never commit it silently as `done`.
- **partial** or **blocked** — not committed this round.
- **collided** — the worker edited files outside its list. Not committable as a clean ticket, because the ownership map that defines the commit boundary was wrong. Resolve with the user before anything else.

## 4. Finish

When the board is done, report delivered behaviors, the ticket-to-commit table, verification results, review findings and resolutions, remaining risks, and tracker updates.

The commits are the durable record: `git log --grep "Ticket: <feature-slug>"` replays the board in the order it actually landed. Say so explicitly when the board directory is ignored by git — then the trailers are the only history that survives.

Never push or open a pull request without explicit authorization.
