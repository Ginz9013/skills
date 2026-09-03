---
name: develop
description: Run the complete interface-first TDD workflow for one change, in a single process — plan a board, implement its tickets one at a time test-first, then review on two axes. Use only when the user explicitly invokes this skill or asks to run their full TDD development workflow. For parallel workers, use develop-parallel instead.
---

# TDD Develop

You are the orchestrator for one change, from requirement to reviewed implementation. You keep the user in control at phase boundaries, and you are the only one who touches git.

You do not implement tickets yourself. That is `implement`'s job, one ticket at a time, so that the same discipline runs whether the work is sequential or dispatched to parallel workers.

## 1. Plan

Use the installed `plan` skill to produce the board. Tell it:

- the change being planned;
- **execution mode: sequential**;
- the publication target — a tracker, or none — after asking the user which, having first checked read-only whether the repository configures one.

If `plan` is not installed, say so plainly and stop rather than improvising a plan. The board's structure is what everything downstream depends on.

Present the board — spec, interfaces, seams, tickets, blocking edges — and get approval before any code is written. Do not begin implementation until the spec, the test seams, and the ticket order are approved.

## 2. Implement, one ticket at a time

Take the first ticket whose blockers are all done. There is no batching here; the whole point of this entry is that exactly one ticket is in flight.

For each ticket, use the installed `implement` skill with **exactly that one ticket**, and give it the complete packet: ticket path, board path, approved seam, write scope, verification commands, baseline, and `isolation: shared working tree`.

The packet must be complete even though you are in the same process. Passing less trains the workflow into a shape that breaks the moment it is dispatched to a worker that cannot see this conversation.

When the report comes back:

- **done** — check the evidence per the rules in `implement`'s execution report reference. Count red-green pairs against acceptance criteria. Missing or inconsistent evidence downgrades the ticket to `unverified`.
- **partial** or **blocked** — report to the user and decide together whether to reshape the ticket, unblock it, or defer it.
- **unverified** — the code may be fine, but the red-green claim is unsupported. Tell the user what is missing and let them choose between redoing the ticket and accepting it knowingly. Do not quietly accept it.

Run the broader suite yourself before moving on. Do not trust a report's test output as verification; it is a claim about a tree you can check directly.

Mark the ticket's status in the board, and in the tracker when publishing.

## 3. Review on two axes

Use `code-review`. Review the diff from an explicit fixed point.

- **Standards** — repository instructions, coding standards, architecture decisions, meaningful code smells. Treat heuristics as judgment calls, and skip what tooling already enforces.
- **Spec** — every acceptance criterion, checked for missing, partial, incorrect, or unrequested behavior.

Keep the axes separate, so clean code cannot hide a wrong implementation and correct behavior cannot hide poor design.

If the change is committed, review against the base commit. If it is not, review the working tree explicitly — `code-review` reads a committed diff, and pointing it at an uncommitted change reports clean without looking at anything, which is a silent failure.

Resolve confirmed blockers, then rerun the affected tests and checks.

## 4. Finish

Report:

- delivered behaviors and any intentional deviations;
- verification commands and their results;
- review findings and resolutions;
- remaining risks and follow-ups;
- tracker updates, when publishing.

Commit only when the user asked for commits. Never push or open a pull request without explicit authorization.

## Escalating to parallel

Switch to `develop-parallel` when the board has several unblocked tickets with disjoint write ownership and the user wants them run concurrently. Do not switch when the board is a linear chain, when the repository is not under git, or when the test suite cannot tolerate concurrent runs — in those cases parallelism costs more than it returns.
