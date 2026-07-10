# Maestro

A Claude Code skill that stops the agent from guessing on complex tasks and turns
task decomposition into a real, cost-aware pipeline instead of ad-hoc subagent calls.

Maestro does two things:

1. **Asks before it assumes.** Any genuine ambiguity — scope, tradeoffs, missing
   requirements — is collected up front and asked via batched questions, not
   guessed at. Clear tasks get zero questions and start immediately.
2. **Delegates like an orchestrator, not a script.** Complex, parallelizable work
   is planned by the best available model at maximum effort, split into
   dependency-ordered waves, and handed to tiered worker agents (haiku → sonnet →
   opus) so cost scales with actual difficulty, not worst-case assumption.

## How it works

```
your prompt
   │
   ▼
Phase 0  Clarify        batched AskUserQuestion rounds (only for real ambiguity)
   │
   ▼
Phase 1  Decompose       obvious split → plan inline
                         complex split → maestro-planner (fable, xhigh effort)
   │
   ▼
Phase 2  Tier            each subtask assigned to the cheapest adequate worker
   │
   ▼
Phase 3  Dispatch        parallel waves, each worker gets a full mergeability
                         contract (context pack, file ownership, output
                         contract, interface pins, narrow verification)
   │
   ▼
Phase 4  Merge + verify  orchestrator integrates results, runs the real
                         build/test suite once, reports back
```

## Files

| File | Role |
|---|---|
| `.claude/skills/maestro/SKILL.md` | The orchestrator's playbook — all five phases |
| `.claude/agents/maestro-planner.md` | Plans decomposition, waves, tiers, and dispatch prompts. `fable`, `xhigh` effort, read-only |
| `.claude/agents/maestro-light.md` | Worker for boilerplate, renames, config, docs. `haiku`, `low` effort |
| `.claude/agents/maestro-worker.md` | Worker for typical features/bugfixes/tests. `sonnet`, `medium` effort |
| `.claude/agents/maestro-heavy.md` | Worker for architecture, tricky algorithms, hard debugging. `opus`, `high` effort |

## Install

Claude Code auto-discovers skills and agents from `.claude/` directories:

- **Global** (all projects): copy the contents of `.claude/skills/maestro/` and
  `.claude/agents/maestro-*.md` into `~/.claude/skills/maestro/` and
  `~/.claude/agents/`.
- **Project-only**: keep them in this repo's `.claude/skills/` and
  `.claude/agents/` — Claude Code picks them up automatically for anyone working
  in this directory.

No build step, no dependencies.

## Usage

Trigger it with `/maestro <task>`, or just describe a big multi-part task — the
skill's description is written to auto-match on "maestro", "orchestrate", and
large/complex requests.

```
/maestro add rate limiting to the API, write tests, and update the docs
```

Maestro will ask anything genuinely unclear (e.g. which endpoints, what limit),
then — if the split is non-trivial — hand it to the planner, dispatch workers in
parallel waves, and merge the result with a real verification pass.

## Design notes

- **Cost discipline**: the expensive model (fable @ xhigh) is reserved for
  planning; execution runs on the cheapest tier that can do the job well.
  Trivial 2–3 subtask splits skip the planner entirely.
- **Token discipline**: workers get a context pack (exact files to read) instead
  of cold-searching the codebase; the shared brief is written once and reused
  across a wave; workers run narrow, targeted verification only — the full
  build/test suite runs exactly once, at merge.
- **Mergeability by construction**: every worker prompt pins exclusive file
  ownership and any shared interfaces before dispatch, so results merge without
  negotiation.
- **`maxTurns` caps** on every agent prevent a stuck worker from looping
  indefinitely.

## Requirements

- Claude Code with subagent support (`.claude/agents/*.md` frontmatter:
  `model`, `effort`, `maxTurns`, `disallowedTools`).
- `model: fable` requires access to Claude Fable 5. If unavailable, change
  `maestro-planner.md`'s `model:` to `opus` — everything else works unchanged.
