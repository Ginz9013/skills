---
name: tdd-implement
description: Implement exactly one ticket from an approved board, test-first, and return an execution report. Use when an orchestration skill assigns a single ticket, or when the user asks to implement one ticket red-green. Scope is one ticket — never a whole board. Never runs a git command that writes.
---

# TDD Implement

You implement **exactly one ticket** and hand it back. Not the board, not the next ticket, not the thing you noticed on the way. Another worker may be editing the rest of this repository right now.

## Input contract

You need all of the following. If any is missing, ask for it rather than guessing — guessing at write scope is how work gets silently overwritten.

- **Ticket** — identifier and file path, or its full text.
- **Board** — path to the spec, so acceptance criteria and seams can be read at the source.
- **Approved seam** — the public seam the tests exercise.
- **Write scope** — the explicit list of files you may edit.
- **Verification** — the focused test, broader suite, and static check commands for this repository.
- **Baseline** — pre-existing failures and type errors you must not repair.
- **Isolation** — shared working tree, or your own worktree. This changes the git rule below.

Read the ticket and the spec before writing anything. Read the repository's instructions, `CONTEXT.md`, and the ADRs the ticket cites; they record decisions you are not free to reopen.

## Boundaries

<hard-rules>
- **Only touch files in your write scope.** If the ticket cannot be finished without a file outside it, stop and report blocked. Do not fix it quickly on the way — in a shared tree there is no conflict marker to catch you, the other worker's edit simply disappears. That list is also the commit boundary, so straying corrupts history, not just the tree.
- **Run no git command that writes** — no `commit`, `add`, `stash`, `checkout`, `restore`, `reset`, `push`. The index is shared. Read-only git (`status`, `diff`, `log`) is fine. The orchestrator does all committing. The single exception: when you were told you are in your own worktree, commit there per slice as described below — still never push.
- **Do not modify existing tests.** They are the safety net. A failing existing test is a finding to report, not an obstacle to edit away. The only exception is when your acceptance criteria name that test.
- **Do not repair the baseline.** Care only about failures in what you touched.
- **Stay inside your shared-resource allocation** — the ports, schemas, and temp directories you were given. Other workers hold the rest.
</hard-rules>

## The loop

Work one behavior at a time. Use `tdd` for the red-green loop discipline. If it is not installed, stop and tell the user to install it — do not run red-green from your own judgment.

1. **Red** — write one failing test through the approved public seam. State the failure you expect, run the test, and confirm the observed failure matches. Capture the output.
2. **Green** — write only enough production code to pass that test. Do not anticipate later slices.
3. Run the focused test and the relevant static check.
4. Take the next behavior, learned from the previous cycle. Repeat.

Never write all the tests and then all the implementation. Tests written after the code they test are shaped by that code, so they pass by construction and catch nothing.

Avoid tests coupled to private methods, internal state, call counts, or mocks of internal collaborators. Expected values come from the spec, a worked example, or another independent source of truth — never from running the code and copying what it printed.

At the end of the ticket, run the broader suite and compare against the baseline.

**In your own worktree**, commit each slice as two commits — the failing test, then the implementation that passes it. This makes the red-green order a fact in history rather than a claim in a report, and the orchestrator will verify it.

**Characterization exception**: when the ticket exists to pin existing behavior before a refactor, write those tests in bulk and expect them green immediately. Say so in the report. Assert observable behavior, never names that the upcoming change will churn.

## Report

Your final message **is** the deliverable — the orchestrator reads it and nothing else, then acts on it. Read [references/execution-report.md](references/execution-report.md) for the required structure and the evidence rules.

A report without per-slice red evidence will be quarantined as `unverified` and will not be committed, so record the evidence as you go rather than reconstructing it afterwards. Never paste output you did not observe.

If you are blocked, say precisely what would unblock you. A blocked report delivered early is worth more than a guess delivered late.
