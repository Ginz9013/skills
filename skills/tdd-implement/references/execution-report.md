# Execution report

The report is the only thing the orchestrator sees. It decides whether your work is committed, quarantined, or handed back.

## Structure

~~~markdown
## Ticket

<NN> <title>

## Status

done | partial | blocked | unverified

## Slices

### Slice 1 — <behavior>

- Test: <test file path>::<test name>
- Expected failure: <what should fail, and why>
- Observed failure: <what actually failed>

```
<verbatim output of the failing run>
```

```
<verbatim output of the same test passing>
```

### Slice 2 — <behavior>

<...>

## Acceptance criteria

- [x] <criterion> — met
- [ ] <criterion> — not met, <reason>

## Files changed

- <path>

## Verification

- <command>

```
<final output: full for focused runs, summary plus failures for the suite>
```

Baseline comparison: <unchanged, or what moved>

## Deviations

<Where the implementation departs from the spec or the approved contract, and why — or None>

## Findings

<What you noticed and did not fix>

## Proposed commit message

<Subject in this repository's convention, plus a body sentence when the why is not obvious>

## Blocked by

<Precisely what would unblock you — omit unless blocked>
~~~

## Evidence rules

Red evidence exists to prove two things: that the discipline actually ran, and that the tests were not written after the code they claim to test. Both failures are invisible in a diff, which is why the evidence is mandatory.

- **Verbatim, not narrated.** Paste the runner's actual output — command, assertion difference, exit code. "The test failed as expected" is not evidence.
- **Expected and observed must match.** A test that fails for an unforeseen reason is a bug in the test, not a red light. Record both and reconcile them.
- **An assertion failure, not a collection error.** `module not found` is acceptable only for the very first slice against a module that does not exist yet, and it must progress to an assertion-level failure before you implement.
- **One red-green pair per behavior slice.** A ticket with five acceptance criteria and one red-green pair did not follow the loop. The orchestrator counts these.
- **In a worktree, history is the evidence.** Per-slice test-then-implementation commits are checkable; report the shas.

## Size

Focused red and green runs: paste in full — they are short. The broader suite: summary line plus any failure sections. Never paste an entire large suite log.

## Status meanings

| Status | Meaning |
| --- | --- |
| `done` | Acceptance criteria met, evidence complete, verification passed. Causes the work to be committed. |
| `partial` | Real progress, not finishable as specified. Say what remains. |
| `blocked` | Cannot proceed. Say precisely what would unblock. |
| `unverified` | The code may be right, but the red-green claim is unsupported. Use it yourself when you know the loop was not followed; the orchestrator also assigns it when evidence is missing or inconsistent. |

Be accurate rather than optimistic. `done` is a claim the orchestrator will re-verify against the whole tree, so an over-claimed ticket costs more than an honest `partial`.
