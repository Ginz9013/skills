---
name: loop-engineering
description: Run the complete interface-first TDD workflow for one change, unattended — wraps plan and implement to plan an approved board and then run every round exactly as develop-parallel does (frontier → batch → dispatch → triage → quarantine → verify → commit → review → close), landing one commit per ticket, without pausing for round-by-round approval. Use only when the user explicitly invokes this skill or asks to run their TDD workflow unattended / hands-off. For a supervised parallel run where you approve each batch, use develop-parallel instead.
---

# Loop Engineering

You are the integrator, same as in `develop-parallel`. You never implement a ticket yourself, you are the only one who touches git, and you never trust a worker's report as verification. The difference here is that once the board is approved, you keep the round loop running on your own — you do not stop to confirm each batch with the user, and an `unverified` report is a policy decision, not a question you ask.

## 0. Preconditions

Check these before planning, same as `develop-parallel`. Each failure is a reason to degrade, not to stop.

- **Git.** The quarantine and per-ticket commit model requires it. Without git, plan here but execute sequentially, as `develop` would.
- **Sub-agent support.** The platform must be able to run workers concurrently. Without it, the whole board is sequential.
- **Test isolation.** Concurrent runs must not fight over ports, database schemas, or temporary directories. When they would, either allocate per worker in the board or run sequentially.
- **One session per repository.** `refs/stash` is shared across the whole repository, worktrees included. A second dispatcher stashing concurrently can make quarantined work vanish from under this one.

Say which preconditions failed and what you are doing about it. Silent degradation is worse than serial execution.

## 1. Plan

Use the installed `plan` skill to produce the board. Tell it:

- the change being planned;
- **execution mode: parallel** — the board must be written to files, because workers cannot see this conversation;
- the publication target — a tracker, or none.

If `plan` is not installed, say so and stop rather than improvising. Get the board approved before any code is written. This approval gate is unchanged from `develop-parallel` — the spec and its tickets still need a human to sign off. Only the round-by-round execution approval below is removed.

## 2. Each round

Read [../develop-parallel/references/batching.md](../develop-parallel/references/batching.md) for frontier, blast-radius, quarantine, and commit rules. Per round:

1. **Frontier** — every ticket whose blockers are done and whose own status is not done or in progress. Empty frontier with nothing in flight means the board is finished.
2. **Batch** — cut the frontier down to tickets with pairwise disjoint write ownership. Everything else queues for a later round.
3. **Dispatch** — once the batch is computed, proceed straight to dispatch. There is no Confirm-with-user step here — that is the one thing this skill removes from `develop-parallel`'s round loop. Spawn one worker per batch ticket, all in a single message, so they actually run concurrently. Read [../develop-parallel/references/delegation-packet.md](../develop-parallel/references/delegation-packet.md) for what each worker gets.
4. **Triage** — read every report before touching git.
5. **Quarantine** — stash everything not committable this round, tagged by ticket.
6. **Verify** — on the quarantined tree, run the full suite and the static checks yourself, before any commit. Parallel work passes in isolation and fails together. Record `HEAD` as the round's base.
7. **Commit** — one commit per ticket, in dependency order, staged by that ticket's file list. Never `git add -A`.
8. **Review** — run the two-axis review against the recorded base, after committing. Apply the review policy — auto-fix small findings, append large findings as new tickets — from [references/autonomy-rules.md](references/autonomy-rules.md) rather than escalating each finding to the user.
9. **Close** — set each landed ticket's status, restore `partial` work, report a ticket-to-commit table, then return to step 1 with the new frontier. Unlike `develop-parallel`, do not stop here to ask whether to continue — keep looping until the board is finished or the circuit breaker fires.

## 3. Judging a report

A worker's report is a claim, not a result. Sort the batch:

- **done** — candidate for commit, *after* its evidence passes the rules in `implement`'s execution report reference. Count red-green pairs against acceptance criteria; check that expected and observed failures match; in worktree mode, check the per-slice commits exist.
- **unverified** — marked `blocked` and not committed, per the `unverified` policy in [references/autonomy-rules.md](references/autonomy-rules.md), not put to the user for a decision. Dependents of that ticket block too; unrelated tickets keep running.
- **partial** or **blocked** — not committed this round.
- **collided** — the worker edited files outside its list. Not committable as a clean ticket, because the ownership map that defines the commit boundary was wrong. This, and every other undecidable or irreversible situation, is judged against the redline in [references/autonomy-rules.md](references/autonomy-rules.md): reversible decisions are made and the loop continues, the listed irreversible categories stop that ticket, block its dependents, and count toward the circuit breaker.

## 4. Autonomy policy

Read [references/autonomy-rules.md](references/autonomy-rules.md) for the full policy this skill applies without asking: the reversible/irreversible redline, the `unverified` → `blocked` handling above, the review auto-fix/escalate policy, and the consecutive-block circuit breaker. That file is the single source for these rules — this document only names the points in the round loop where each one is invoked.

## 5. Finish

When the frontier is empty with nothing in flight, or the circuit breaker in [references/autonomy-rules.md](references/autonomy-rules.md) fires, stop the loop. Read [references/summary-report.md](references/summary-report.md) and write `.scratch/<feature-slug>/summary.md`: run metadata, the ticket-to-commit table, verification results, review findings and how they were resolved, anything left needing a human decision, whether the circuit breaker fired, and suggested next steps.

The commits are the durable record: `git log --grep "Ticket: <feature-slug>"` replays the board in the order it actually landed. Say so explicitly when the board directory is ignored by git — then the trailers are the only history that survives.

Never push or open a pull request without explicit authorization.
