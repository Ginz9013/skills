# Board artifact structures

Read this when drafting the spec or the tickets. Adapt headings to the repository's existing convention; preserve the information, not the exact formatting.

## Spec

```markdown
# <Feature or change>

## Problem

<Current problem and why it matters>

## Outcome

<Observable result when complete>

## Scope

- <Included behavior>

## Non-goals

- <Explicitly excluded behavior>

## Domain decisions

- <Agreed term, rule, or invariant>

## Design contract

### <Module>

- Responsibilities: <owned behavior>
- Non-responsibilities: <behavior owned elsewhere>
- Interface: <inputs, outputs, errors, invariants, side effects>
- Seam: <public observation/replacement point>
- Dependencies: <production and test strategy>

## Acceptance criteria

- [ ] <Observable behavior>

## Test strategy

- Seam: <approved public seam>
- Test type: <integration, contract, unit, end-to-end>
- Dependency handling: <real, local substitute, adapter, external mock>

## Verification

- Focused: <command for a single test file>
- Suite: <command for the broader regression suite>
- Static: <typecheck or lint command>

## Baseline

Measured at <date/sha>. Do not repair these unless a ticket says to.

- <file>: <N failing tests or type errors>

## Risks and deferred questions

- <Risk or intentionally deferred decision>
```

## Ticket

```markdown
# <NN> <Observable behavior>

## Outcome

<One end-to-end capability this ticket delivers>

## Acceptance criteria

- [ ] <Behavior observable through the public interface>

## Test seam

<Approved public seam and expected test level>

## Write ownership

- <path this ticket is expected to edit>

Anything outside this list belongs to someone else. Mark as `unbounded` when
the set cannot be established; an unbounded ticket runs alone.

## Shared resources

<Port, database schema, temp directory this ticket needs — or None>

## Blocked by

<Ticket numbers, or None>

## Status

todo | in-progress | done | blocked | unverified

## Done when

- A failing behavior test was observed before implementation, per slice.
- Focused tests and relevant static checks pass.
- The broader regression suite appropriate to the change passes.
- Standards and spec review blockers are resolved.
```

## Slicing checks

Reject or reshape a proposed ticket when it is primarily:

- database or schema work with no observable behavior;
- API plumbing with no usable capability;
- UI scaffolding with no working interaction;
- "write all the tests" before implementation;
- a refactor unrelated to the requested outcome.

Infrastructure belongs inside the first behavior slice that needs it.

## Write ownership checks

- Two tickets that name the same file cannot run in the same parallel batch. That is a fact to record, not a reason to redraw the slice dishonestly.
- A mechanical change fanning across the codebase — a rename, a shared type change — owns everything. Mark it `unbounded`.
- Ownership is also the commit boundary in parallel runs. A wrong list corrupts history, not just the working tree.
