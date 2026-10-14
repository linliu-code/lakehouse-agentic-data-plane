# RFC 0001: Agent cost is a data plane problem

- **Status:** Draft
- **Author:** Lin Liu
- **Created:** 2026-10-14

## Summary

When an agent answers a question or completes a task over lakehouse data, the bill is
driven less by model tokens than by the data work underneath: scans, joins, repeated
exploratory queries, and data copied out of the lakehouse into the agent's context.
This RFC argues that agent cost should be treated as a data plane concern — measured,
bounded, and optimized where the data lives — rather than as a prompt- or model-tuning
concern.

## Motivation

Agents differ from human analysts and from scheduled pipelines in three ways that matter
for cost:

1. **They explore.** An agent issues many speculative queries (schema probes, samples,
   retries after a wrong guess) before the one that answers the task.
2. **They repeat.** Similar tasks re-derive the same intermediate results, because the
   agent keeps no durable memory of what it already computed.
3. **They pull data toward the model.** Results are materialized into context windows,
   which means data leaves the engine that could have filtered, aggregated, or skipped it.

Each of these multiplies data-plane work per task. None of them is addressed by a
cheaper model or a shorter prompt.

## Thesis

Agent cost = model cost + data cost, and for data-heavy tasks the data term is the one
that scales with table size and task count. It is therefore controlled by data-plane
mechanisms:

- **Read less:** pruning, file/row-group skipping, indexes, and pushdown applied to
  agent-issued queries, not only to hand-written ones.
- **Recompute less:** reuse of intermediate results across agent steps and across tasks
  (incremental reads, materialized and cached results with lakehouse-native freshness).
- **Move less:** push computation to the engine and return the smallest answer the agent
  needs, instead of rows into context.
- **Bound it:** per-task data budgets (bytes scanned, queries issued) enforced by the
  data plane, with attribution so cost is visible per agent and per task.

## Non-goals

- Choosing or tuning a specific model.
- Prompt engineering techniques for reducing token count.
- A specific query engine or table format; the argument applies to the lakehouse layer
  generally.

## Open questions

- How much of a typical agent task's cost is data cost versus model cost? This RFC does
  not yet have measurements; quantifying the split is the first follow-up.
- What is the right unit for a per-task data budget — bytes scanned, compute-seconds, or
  queries?
- Where should reuse live: in the engine's cache, in the table format's metadata, or in
  an agent-facing layer above both?

## Next steps

Follow-up RFCs will take each mechanism above (read less, recompute less, move less,
bound it) and propose a concrete design.
