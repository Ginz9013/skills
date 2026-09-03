# Autonomy rules

Lookup table for what a worker or the orchestrator may decide alone,
mid-round, and what forces a hard stop. This document does not restate
frontier, batch, quarantine, or commit mechanics — those stay owned by
`batching.md`.

## Reversible-decision default

Default to autonomous. Any action whose consequences `git revert` can
undo inside this repository is decided on your own, choosing the best
available option, so the round keeps moving. This explicitly includes:

- Editing pre-existing files, not just creating new ones.
- Adjusting a cross-module interface.

Do not stop to ask about a reversible decision. Make the call, record it
in the ticket's report if relevant, and continue.

## The five hard-stop categories

Any one of these stops the ticket, even though the round otherwise
continues:

<hard-rules>
1. **Out-of-repo deletion or overwrite.** Deleting or overwriting
   anything outside the repository's version control (databases, user
   files, cloud resources).
2. **Force-push or CI/CD or branch-protection changes.** Force-pushing,
   changing branch-protection rules, or editing CI/CD pipeline
   configuration.
3. **Secrets, credentials, permissions, or a real external side effect.**
   Touching secrets, credentials, or permissions, or sending a request to
   an external service with a real side effect.
4. **Write ownership that cannot be determined.** The existing mechanical
   limit from `batching.md`: a ticket whose write ownership genuinely
   cannot be established.
5. **A defect in the approved board or spec itself.** Its acceptance
   criteria contradict the code as it actually exists, or otherwise
   cannot be satisfied as written.
</hard-rules>

For every category above:

- **That ticket:** stop work on it immediately. Do not commit it.
- **Its dependents:** blocked — they cannot enter the frontier until the
  blocking ticket is resolved.
- **Unrelated tickets in the same round:** continue running; a hard stop
  on one ticket is not a hard stop on the round.

## `unverified` policy

`develop-parallel` puts an `unverified` report to the user for a
decision. `develop-loop` replaces that rule with this one, for this
skill only:

- Mark the ticket `blocked`.
- Do not commit it.
- Block its dependents, same as a hard-stop category.
- Unrelated tickets in the same round continue running.

## Review policy

Review runs after every commit, not just at the end of the round:

- **Small findings** — fix automatically, landed as an extra commit. The
  run continues without stopping.
- **Large findings** — do not fix inline. Append a new ticket to the
  board describing the finding, and continue the run without stopping
  for the user.

## Circuit breaker

Track consecutive tickets stopped under a hard-stop category (the five
above), across the round, regardless of batch.

- Default `N = 3`.
- When `N` consecutive tickets have been stopped this way, end the round
  loop early — do not compute another frontier or dispatch another
  batch — and proceed straight to producing the summary.
- The caller may override `N` when invoking the skill.
- A ticket resolved successfully resets the consecutive count to 0.
