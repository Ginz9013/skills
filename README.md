# tdd-workflow

An interface-first TDD development workflow for coding agents, packaged as a plugin of five composable skills.

Plan a board, implement one ticket at a time test-first, and review on two independent axes — running the tickets sequentially in one process, or dispatching them to parallel workers, without maintaining two copies of the method.

## Why it is split this way

Most workflow skills are one long document. That works until you want a second execution strategy, at which point you fork the whole thing and the two copies drift.

Here the seam is cut at **one ticket → one verified diff**:

```
tdd-plan       board: spec + interfaces + seams + tickets + write ownership
                 │
     ┌───────────┴───────────┐
tdd-develop            tdd-develop-parallel
  one ticket at a time     collision-safe batches → sub-agents
     └───────────┬───────────┘
                 │
tdd-implement   exactly one ticket → red-green → execution report
```

`tdd-implement` does not know whether it is the only worker or one of four. That is what lets both entry points share it verbatim. The entry points stay thin — they own git, approval gates, and integration; they own no engineering method at all.

## The skills

| Skill | Kind | Scope | Touches git |
| --- | --- | --- | --- |
| `tdd-develop` | user-invoked | Entry: plan, then implement tickets one at a time, then review | yes |
| `tdd-develop-parallel` | user-invoked | Entry: plan, then dispatch batches to workers, one commit per ticket | yes |
| `tdd-plan` | model-invoked | Produce an approved board | no |
| `tdd-implement` | model-invoked | Implement **exactly one** ticket, test-first | no |
| `loop-engineering` | user-invoked | Entry: plan, then run every round unattended, one commit per ticket, autonomous within the reversible-decision redline | yes |

`agents/tdd-worker.md` is an optional Claude Code sub-agent definition. It is a thin shell that loads `tdd-implement`; the discipline itself lives in the skill, so nothing is duplicated per platform.

## Core ideas

**The board is the contract.** `tdd-plan` produces a spec, the public interfaces, the agreed test seams, vertical-slice tickets, and — always, regardless of execution mode — the **write ownership** of each ticket. It also measures the repository's current baseline of failing tests and type errors, so a worker never "helpfully" repairs something outside its scope.

**Write ownership does double duty.** During a parallel round it is the collision boundary: two tickets naming the same file cannot run together, because in a shared working tree the loser's edit disappears with no conflict marker. Afterwards it is the commit boundary — which is why a wrong list corrupts history, not just the tree.

**Red evidence is mandatory.** An execution report must carry, per behavior slice, the expected failure, the observed failure, and the verbatim output of the failing run followed by the passing one. This guards two failures that are invisible in a diff: an implementation produced without the red-green loop at all, and tests written after the code they claim to test. A report without it is quarantined as `unverified` and is not committed. Under worktree isolation the evidence is stronger still — per-slice commits make the ordering a fact in git history rather than a claim in prose.

**Reports are claims, not results.** The orchestrator re-runs the full suite itself on the quarantined tree before any commit. Parallel work passes in isolation and fails together.

**Publication and storage are independent.** Whether a tracker is used decides where the board is announced. Whether the board is written to files decides where it lives during execution — and parallel runs require files, because workers do not share the orchestrator's conversation.

**Autonomy has a redline.** `loop-engineering` runs `tdd-develop-parallel`'s round loop unattended, deciding on its own except when an action is irreversible outside git — deleting or overwriting anything outside version control, touching secrets or credentials, force-pushing, or discovering that the board itself is defective. Anything short of that line, the loop keeps moving; anything past it, that ticket stops and its dependents block. The full redline is recorded in [autonomy-rules.md](skills/loop-engineering/references/autonomy-rules.md).

## Install

### Claude Code

As a plugin — install this repository as a marketplace or point `--plugin-dir` at it. That bundles the five core skills, the five vendored dependencies, and the worker agent together, which avoids the main failure mode of split skills: an entry point that is installed while the skills it names are not.

Or copy individual skills into `.claude/skills/` (project) or `~/.claude/skills/` (user). If you do, install all five core skills plus the five vendored ones (or resolve their symlinks first — see [Vendored dependencies](#vendored-dependencies)); the entry points name them explicitly and will stop rather than improvise when one is missing.

### Codex

Copy each directory under `skills/` into `~/.codex/skills/`. Each carries an `agents/openai.yaml` describing its interface and whether implicit invocation is allowed.

### Gemini CLI

Not yet supported — see below.

## Platform support

| Platform | Skills | Sub-agents | Status |
| --- | --- | --- | --- |
| Claude Code | yes | yes, with worktree isolation | both entry points verified |
| Codex | yes | no delegation primitive in older CLI builds | sequential entry only; unverified |
| Gemini CLI | not in older builds | no | unsupported for now |

Composition between skills is **model-level, not runtime-level**: there is no typed call, no return value, and no invocation exception on any of these platforms. A skill names another skill and the model loads it. The architecture is therefore built as an artifact protocol — each stage produces a document the next stage can read — rather than as a call stack.

Probe testing on Claude Code found the honest-failure behavior reliable: when a named skill is absent, the model reports it as missing instead of silently improvising, even when the calling skill does not ask it to report status. That is model behavior, not a runtime guarantee, which is exactly why the red-evidence rule stays.

## Preconditions for parallel execution

`tdd-develop-parallel` checks these and degrades to sequential rather than failing:

- the repository is under git — quarantine and per-ticket commits depend on it;
- the platform can run sub-agents concurrently;
- the test suite tolerates concurrent runs, or the board allocates ports, schemas, and temp directories per worker;
- only one dispatch session runs per repository — `refs/stash` is shared across the whole repository, worktrees included.

It says which precondition failed. Silent degradation is worse than serial execution.

## Known limitations

- The rule that a worker must never run a writing git command is enforced by instruction, not by tooling: a sub-agent's tool whitelist cannot restrict git, because git runs through the shell. Enforcing it mechanically needs a `permissions.deny` rule in your own settings.
- The Codex and Gemini paths are unverified against current CLI builds.
- Parallel execution has not yet been exercised end-to-end on a repository with a concurrent-safe test suite.
- `loop-engineering` reads `tdd-develop-parallel`'s `batching.md` and `delegation-packet.md` by relative path rather than copying them, so it silently breaks if those files are renamed or moved without updating `loop-engineering/SKILL.md`.

## Vendored dependencies

This workflow delegates to five skills from [Matt Pocock's skills](https://github.com/mattpocock/skills) — `grilling`, `domain-modeling`, `codebase-design`, `tdd`, `code-review` — for the interview discipline, glossary/ADR work, interface design, the red-green loop, and standards/spec review respectively.

They are vendored, not optional. The upstream repository is merged as-is, untouched, into [vendor/mattpocock-skills/](vendor/mattpocock-skills/) via `git subtree`, and `skills/grilling`, `skills/domain-modeling`, `skills/codebase-design`, `skills/tdd`, `skills/code-review` are symlinks into that tree (`skills/<name>` → `vendor/mattpocock-skills/skills/<category>/<name>`). Cloning or downloading this repository gets the real files; there is no separate install step and no silent "skill not found" fallback in `tdd-plan`, `tdd-implement`, or `tdd-develop`.

To pull upstream updates:

```
git subtree pull --prefix=vendor/mattpocock-skills mattpocock-skills main --squash
```

(add the remote once with `git remote add mattpocock-skills https://github.com/mattpocock/skills.git`). If upstream renames or moves one of the five skills, its symlink target changes and needs updating by hand.

Codex installs (see above) copy directories under `skills/` verbatim — resolve the five symlinks into real files first (e.g. `cp -rL`), since `~/.codex/skills/` won't have this repo's `vendor/` alongside it.

## Credit

The composition model — thin user-invoked orchestration over reusable model-invoked disciplines — follows [Matt Pocock's skills](https://github.com/mattpocock/skills).
