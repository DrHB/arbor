---
name: stigmergic-session
description: Run a shared-board multi-agent session for planning, critique, prioritization, triage, or synthesis without a strategic supervisor. Use this skill when the user wants multiple roles to converge through a structured substrate, or wants to compare board-based coordination with a normal supervisor workflow.
---

# Stigmergic Session

## Overview

Use this skill to run a small multi-agent session where roles coordinate through a shared structured board instead of a central planner. Prefer it for fuzzy, multi-perspective tasks where partial discoveries should influence later work.

Plain-English intuition:
- In an ant colony, one ant does not hold the master plan
- Ants leave pheromone trails in the environment
- Other ants react to those trails
- Useful trails get reinforced
- Weak trails fade

This skill uses the same pattern for agent work:
- the shared board is the environment
- signals on the board are the trails
- roles do not directly supervise each other
- roles read the board, act on it, and leave traces for later roles
- convergence happens because strong signals survive repeated contact

Good fits:
- planning and scope decisions
- design critique
- prioritization
- bug or incident triage
- synthesis from fragmented notes

Bad fits:
- exact math
- deterministic algorithmic optimization
- simple factual lookup
- one-shot tasks where a normal single-agent answer is enough

## Core Model

A stigmergic session has six moving parts:

1. A task
2. `N` roles, usually 3-5
3. A shared substrate
4. A tick loop
5. Deterministic scoring rules
6. A convergence and extraction rule

The important distinction from a supervisor workflow:
- A scheduler may still exist to decide turn order
- The scheduler is dumb
- No role owns the plan
- Coordination happens through traces left on the shared board

Map the analogy directly:
- ant colony -> role set
- pheromone field -> `board.json`
- pheromone trail -> signal record
- reinforcement -> `reinforce`
- trail conflict or avoidance -> `object`
- evaporation -> deterministic decay

## Standard Substrate

Standardize on:
- `board.json` for the current state
- optional `signals.jsonl` for append-only history

Use this minimum signal shape inside `board.json`:

```json
{
  "tick": 0,
  "signals": [
    {
      "id": "sig-001",
      "kind": "idea",
      "content": "Build the audit substrate first",
      "status": "open",
      "support": 0.5,
      "objection": 0.0,
      "strength": 0.5,
      "created_by": "builder",
      "last_touched_tick": 0,
      "links": []
    }
  ]
}
```

Use `kind` values only as needed. Default set:
- `idea`
- `risk`
- `question`
- `decision`
- `evidence`

Use `status` values:
- `open`
- `resolved`
- `discarded`

## Role Selection

Choose the minimum set of roles needed for the task. Default role pool:

- `builder`: pushes toward something concrete and shippable
- `critic`: looks for weak assumptions, overreach, and failure modes
- `operator`: focuses on observability, reliability, and constraints
- `user-advocate`: focuses on end-user value and usability
- `synthesizer`: finds bridges, merges duplicates, and proposes compromise
- `investigator`: looks for missing facts, root causes, and evidence gaps

Default session size:
- 3 roles for simple tasks
- 4-5 roles for richer planning or triage

## Allowed Actions Per Turn

Each role must read the current board before acting. A role may do exactly one primary action per turn:

- `create`
- `reinforce`
- `object`
- `merge_duplicate`
- `resolve`
- `noop`

Recommended action output shape:

```json
{
  "action": "reinforce",
  "target_id": "sig-001",
  "content": "",
  "rationale": "This remains the most leverageful option after reading the board."
}
```

Rules:
- Roles do not directly command each other
- Roles act only on the substrate
- If a role has nothing useful to add, use `noop`
- If two signals clearly describe the same thing, prefer `merge_duplicate`

## Deterministic Scoring Rules

Do not let the model invent numeric strengths freely. Models choose actions; the workflow updates numbers deterministically.

Default v1 scoring:
- `create`: initialize `support = 0.5`
- `reinforce`: `support += 0.2`
- `object`: `objection += 0.15`
- `merge_duplicate` with independent support: `support += 0.1`
- untouched signal per tick: apply `decay += 0.05`

Compute:

```text
raw_strength = support - objection - decay_component
strength = clamp(0, 1, raw_strength)
```

Where:
- `decay_component = max(0, current_tick - last_touched_tick) * 0.05`
- `last_touched_tick` updates whenever a signal is created, reinforced, objected to, merged, or resolved

Practical effect:
- repeated support strengthens a signal
- objections weaken it
- ignored signals fade

## Session Workflow

Follow this workflow exactly:

1. Classify whether the task is a good fit
2. Choose 3-5 roles
3. Initialize `board.json`
4. Run ticks
5. After each full tick, apply deterministic scoring updates
6. Stop on convergence or max ticks
7. Extract strongest unresolved signals into the final answer

Default stop rules:
- `max_ticks = 5`
- stop early if no signal changes by more than `0.1` across two consecutive ticks

Default extraction rule:
- Final answer should include:
  - strongest surviving `idea` or `decision` signals
  - strongest open `risk` signals
  - any unresolved `question` signals worth carrying forward

## Runtime Mode

Prefer subagents when available:
- one role per subagent
- all roles read the same board snapshot
- merge actions back into the substrate

If subagents are unavailable, use serial fallback:
- simulate each role explicitly in sequence
- keep role identity stable across ticks
- do not collapse into a single blended voice

The serial fallback is a required behavior, not an optional shortcut.

## Worked Example

Task:
- `What should we build first for Arbor?`

Roles:
- `builder`
- `critic`
- `operator`

Tick 1:
- builder `create`: `Build audit substrate first`
- critic `create`: `Risk: audit-only may feel generic`
- operator `create`: `Add replay from checkpoints`

After deterministic scoring:
- `Build audit substrate first` -> `0.5`
- `Risk: audit-only may feel generic` -> `0.5`
- `Add replay from checkpoints` -> `0.5`

Tick 2:
- builder `create`: `Minimal read-only signal view`
- critic `object` on `Build audit substrate first`
- operator `reinforce` `Add replay from checkpoints`

After deterministic scoring:
- `Build audit substrate first` weakens slightly
- `Add replay from checkpoints` becomes stronger
- `Minimal read-only signal view` enters as a new contender

Likely extraction after a few ticks:
- build audit substrate first
- include replay
- include a small read-only visualization
- do not build a full live conductor UI yet

## Output Contract

When finishing a session, always produce:

1. `Top recommendations`
2. `Top risks`
3. `Unresolved conflicts`
4. `Why the board converged there`

Keep the final output grounded in the board state, not just a fresh freeform opinion.
