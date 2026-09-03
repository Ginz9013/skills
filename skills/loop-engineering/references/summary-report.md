# Summary report template

`loop-engineering` writes one `summary.md` per run, at
`.scratch/<feature-slug>/summary.md`. It is produced once, when the run
ends — either because every ticket on the board reached a terminal
status, or because the circuit breaker fired (see `autonomy-rules.md`).

The summary is not a replacement for the board. It exists so that
whoever reads it afterward — including the same user hours or days
later — can understand what happened in a few minutes, without
re-reading the board or the commit log. Keep every section short: the
per-ticket table plus the sections below should be read-in-minutes, not
a re-derivation of the full board. If a section would need more than a
few lines per item, that detail belongs in the board or the commit
history, not here.

## Fields

### Run metadata

A short block, filled in once per run:

- **Feature slug** — matches the board's `.scratch/<feature-slug>/`
  directory.
- **Board** — path to `spec.md`.
- **Start time** / **End time** — when the run began and ended.
- **Total ticket count** — how many tickets were on the board when the
  run started.
- **Circuit breaker fired** — yes/no. If yes, name the ticket that
  tripped it and how many consecutive stops preceded it.

### Per-ticket table

One row per ticket on the board, no exceptions — every ticket must land
somewhere in this table, even ones the run never reached.

Columns, at minimum:

| Column | Notes |
| --- | --- |
| Ticket id | As it appears on the board. |
| Title | Short, from the board. |
| Status | One of: `done`, `blocked`, `blocked (dependent)`, `quarantined-unverified`, `partial`, `skipped`. |
| Commit sha | The commit that landed it, if any. Blank otherwise. |
| Reason | One line, only when status is not `done`. Blank for `done` rows. |

Status meanings, for reference:

- `done` — landed, verified, committed.
- `blocked` — a worker reported `blocked` or hit a hard-stop redline
  category; nothing committed.
- `blocked (dependent)` — never attempted, because a ticket it depends
  on is `blocked` or `quarantined-unverified`.
- `quarantined-unverified` — a worker's report lacked red-green
  evidence; left quarantined per `autonomy-rules.md`'s policy, not
  committed.
- `partial` — real progress landed, but the ticket was not finishable
  as specified.
- `skipped` — deliberately not attempted (e.g. superseded by another
  ticket, or dropped as out of scope during the run).

### Review findings

What the post-commit review pass found and how it was handled, per
`autonomy-rules.md`'s review policy:

- **Auto-fixed** — small findings fixed automatically. List each with
  the commit sha of the fix.
- **Escalated** — larger findings turned into a new ticket appended to
  the board. List each with the new ticket's id and its resulting
  status (from the per-ticket table above).

If review found nothing worth reporting, say so in one line rather than
omitting the section.

### Needs your decision

Every ticket that hard-stopped, one entry each:

- **Ticket id and title.**
- **Which of the five `autonomy-rules.md` hard-stop categories it hit**
  (out-of-repo destructive action; force-push/branch-protection/CI
  config; secrets/credentials/permissions/external side effects;
  undeterminable write ownership; defective board or spec).
- **What unblocking it requires** — the specific decision or input only
  the user can supply.

If nothing hard-stopped, say so in one line rather than omitting the
section.

### Next steps

A short bulleted list of suggested actions — e.g. resolve a specific
`blocked` ticket, decide on a flagged risk, re-run the board after
fixing the spec defect that stopped it. Not a restatement of the board;
just what to do next given this run's outcome.

## Example

The following is a short, plausible example of a filled-in summary for
a hypothetical `add-rate-limit` run where the circuit breaker did not
fire.

---

### Run metadata

- **Feature slug:** `add-rate-limit`
- **Board:** `.scratch/add-rate-limit/spec.md`
- **Start:** 2026-09-03 14:02
- **End:** 2026-09-03 14:41
- **Total tickets:** 5
- **Circuit breaker fired:** no

### Per-ticket table

| Ticket id | Title | Status | Commit sha | Reason |
| --- | --- | --- | --- | --- |
| 01 | Add token-bucket limiter | done | `a1b2c3d` | |
| 02 | Wire limiter into middleware | done | `d4e5f6a` | |
| 03 | Add per-route override config | blocked | | write scope required editing `config/schema.json`, which is outside version control's rename history and ambiguous per `autonomy-rules.md` category 4 |
| 04 | Add limiter metrics | blocked (dependent) | | depends on 03 |
| 05 | Document rate-limit behavior | quarantined-unverified | | report had implementation but no red-evidence for slice 2 |

### Review findings

- **Auto-fixed:** unused import in `middleware/limiter.ts`, fixed in
  commit `d4e5f6a` (bundled with ticket 02's landing commit).
- **Escalated:** inconsistent error message casing across the limiter
  module, turned into ticket `06` (status: `skipped` — out of scope for
  this run, left for the next planning pass).

### Needs your decision

- **03 — Add per-route override config:** hit category 4
  (write ownership genuinely undeterminable). Unblocking requires
  deciding whether `config/schema.json` belongs to this ticket or a
  separate schema-migration ticket, then re-running the board.
- **05 — Document rate-limit behavior:** report was marked
  `quarantined-unverified`. Unblocking requires reviewing the diff by
  hand and either re-running the worker with the red-green loop
  enforced, or accepting the change manually.

### Next steps

- Decide write ownership for `config/schema.json`, then re-run tickets
  03 and 04.
- Review ticket 05's diff and either re-dispatch it or accept it
  manually.
- Consider ticket 06 (error message casing) for the next planning
  round.
