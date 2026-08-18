# Delegation packet

Each worker gets one packet. It must stand alone: a worker does not share your conversation, has not read the board, and must not be assumed to have loaded any skill you loaded.

Use the project's TDD worker agent type when one is defined. Otherwise spawn a general-purpose sub-agent — the packet is what carries the discipline either way.

## Shape

```markdown
Use the installed `tdd-implement` skill to implement exactly ticket <NN>.
Read its instructions before you begin. Implement that ticket and nothing else.

Ticket:        <path to the ticket file>
Board:         <path to the spec>
Seam:          <the approved public seam these tests exercise>

Write scope — you may edit only these files:
- <path>
Every other file in this repository belongs to another worker running right
now. If you cannot finish without a file outside this list, stop and report
blocked.

Shared resources allocated to you:
- <port / database schema / temp directory, or None>

Verification:
- Focused: <command>
- Suite:   <command>
- Static:  <command>

Baseline — pre-existing failures you must not repair:
- <file>: <N failures or type errors>

Isolation: shared working tree | your own worktree at <path>

Run no git command that writes. The index is shared; the orchestrator does all
committing.  [worktree variant: Commit each slice in your worktree — the
failing test, then the implementation. Never push.]

Your final message is the deliverable. Follow the execution report structure
from the `tdd-implement` skill, including per-slice red evidence. A report
without it will be quarantined and not committed.
```

## Why each field is load-bearing

- **Name the skill explicitly.** A sub-agent does not inherit what you loaded. Naming it is what makes the discipline arrive; omitting it produces a plausible implementation with no red-green loop, and that failure is invisible in the diff.
- **Say "exactly ticket NN".** Without the boundary, a worker that finishes early helpfully starts the next one — inside another worker's files.
- **Write scope is doing double duty.** It is the collision boundary during the round and the commit boundary afterwards. Assign it deliberately rather than approximately.
- **Baseline measured now, not remembered.** Without it, a worker sees a pre-existing red and repairs it out of good intentions — the hardest kind of scope violation to notice, because it looks like care.
- **Isolation changes the git rule.** In a shared tree, writing git is forbidden. In a worktree, per-slice commits are required, because they turn the red-green order from a claim into a verifiable fact.

## What not to put in the packet

- The whole spec. Give the path; the worker reads what it needs. Long packets get truncated, and a truncated packet fails silently.
- Ticket status bookkeeping. Only you write to the board.
- Anything about other tickets beyond "those files are not yours".
