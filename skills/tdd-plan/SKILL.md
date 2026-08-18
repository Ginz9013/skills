---
name: tdd-plan
description: Produce an approved board — spec, public interfaces, test seams, vertical-slice tickets, write ownership, and verification commands — for one feature, change, or bug fix. Use when an orchestration skill asks for a plan before implementation, or when the user wants work planned before any code is written. Never writes production code.
---

# TDD Plan

Produce **one board**: an approved specification plus an ordered set of vertical-slice tickets that `tdd-implement` can execute one ticket at a time, in a single process or dispatched to parallel workers.

You do not write production code, do not edit tests, and do not run any git command that writes.

## Input contract

Establish these before doing anything else. Ask the caller when unstated.

- **The change** being planned.
- **Execution mode** — sequential or parallel. Parallel requires the board to be written to files, because workers do not share this conversation.
- **Publication target** — an issue tracker, or none.

Publication and storage are independent. Publication decides where the board is *announced*; storage decides where it *lives during execution*.

|  | sequential | parallel |
| --- | --- | --- |
| no tracker | conversation (default) or files | **files, required** |
| tracker | conversation + published mirror | **files, required** + published mirror |

When the board is stored as files, default to `.scratch/<feature-slug>/` — one `spec.md` plus one file per ticket — and let the user override the location. Do not modify `.gitignore`; if the directory is already ignored, say so, because then ticket status lives only on disk.

## Phase 1: Discover

Clarify the goal and sharpen the domain model. Use `grilling` for the interview discipline and `domain-modeling` for glossary and ADR work when they are installed; otherwise do this directly.

1. Read the repository's instructions, `CONTEXT.md`, ADRs, and the smallest useful portion of the code.
2. Establish the desired outcome, users, observable behavior, constraints, non-goals, edge cases, and unresolved decisions.
3. Ask focused questions in rounds. Keep an explicit frontier of unresolved decisions; stop when none remain or the user defers them deliberately.
4. Use the repository's own domain terms. Challenge overloaded or ambiguous ones.

Do not invent product decisions. Investigate facts; ask the user to decide tradeoffs.

When a question cannot be settled by conversation, pause and recommend the smallest useful detour: research for factual uncertainty, a throwaway prototype for behavior or UI uncertainty, a questionnaire when someone else owns the knowledge.

## Phase 2: Design the interface and test seams

Design before specifying. Use `codebase-design` when installed.

1. Identify the modules that own the behavior.
2. State each module's responsibilities and explicit non-responsibilities.
3. Define the smallest useful public interface: inputs, outputs, errors, invariants, side effects, relevant performance constraints.
4. Place seams where behavior genuinely varies or must be observed. Do not add an adapter for a hypothetical future variation.
5. Classify each dependency as in-process, locally substitutable, owned remote, or truly external, and choose real implementations or test adapters accordingly.
6. Agree with the user on the public seams the tests will exercise.

The output is a behavioral contract, not an implementation blueprint. Do not predesign private functions, class hierarchies, or file layouts.

Seam placement decides whether tickets can run in parallel at all: slices that share a seam usually share files. When the caller asked for parallel execution, prefer a decomposition whose slices own disjoint files — but never distort the design to manufacture parallelism, and say so plainly when the work is inherently serial.

## Phase 3: Synthesize the spec

Convert the approved conversation and design into a spec without reopening the interview. Read [references/artifacts.md](references/artifacts.md) for the structure.

Present the draft for approval before continuing.

## Phase 4: Cut vertical-slice tickets

Break the approved spec into tracer-bullet tickets. Read [references/artifacts.md](references/artifacts.md) for the ticket structure and the slicing checks.

- Each ticket delivers one observable behavior through an agreed public seam.
- Prefer an end-to-end thin slice over horizontal infrastructure or test-only work.
- Declare blocking edges explicitly and order blockers first.
- A small change may produce one ticket. Do not manufacture ceremony.

Every ticket carries **write ownership** — the files it is expected to edit — regardless of execution mode. Sequential runs ignore it; parallel runs depend on it for collision safety and as the commit boundary. Estimate honestly and mark a ticket as unbounded when you cannot tell; an unbounded ticket is still a valid ticket, it simply cannot be batched.

## Board output

The board must carry everything a worker needs without access to this conversation:

- approved spec and acceptance criteria;
- public interfaces and agreed test seams;
- ordered tickets with blocking edges;
- write ownership per ticket;
- shared-resource constraints — ports, database schemas, temporary directories — when parallel;
- **verification commands** for this repository: focused tests, broader suite, typecheck or other static checks;
- **known baseline** — pre-existing failures and type errors, measured now, so a worker does not "fix" them and stray outside its scope;
- publication target and tracking mode.

Measure the baseline rather than assuming it is clean.

Present the complete board for approval. Publish only after approval, and only to the agreed target. Do not create tracker records, commit, push, or open a pull request without explicit authorization.
