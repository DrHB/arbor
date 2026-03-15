# Arbor

Arbor explores stigmergic coordination workflows for LLM agents using shared structured substrates instead of a central supervisor.

## The Idea

The intuition comes from ants.

Ant colonies often solve coordination problems without one ant acting as the boss. An ant finds food, leaves a pheromone trail, and other ants react to that trail. Useful trails get reinforced. Weak or irrelevant trails fade. Over time, the colony converges on paths that work, even though no single ant planned the whole system.

Arbor applies the same pattern to agent workflows:
- the shared board is the pheromone field
- signals on the board are the trails
- agent roles read the board, then add, reinforce, object to, or resolve signals
- strong signals survive because multiple roles keep touching them
- weak signals decay because nobody reinforces them

The goal is not "no orchestration at all." A dumb scheduler may still decide turn order. The key claim is narrower: the strategy should emerge from the shared substrate, not from one supervisor agent that decides everything up front.

This makes Arbor most interesting for tasks like:
- planning and scope decisions
- design critique
- prioritization
- triage and root-cause exploration
- synthesis from fragmented notes

Today this repo contains:
- a first Codex/Claude-compatible skill for running shared-board multi-agent sessions

This repo may later grow to include:
- benchmark prompts
- optional helpers or scripts
- broader Arbor runtime experiments
