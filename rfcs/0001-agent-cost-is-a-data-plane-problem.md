# RFC 0001: Agent cost is a data plane problem

- **Status:** Draft (v1)
- **Author:** Lin Liu
- **Created:** 2026-10-14

## Problem

A single agent task fans out into N queries and scans. Nobody sees what that cost until
the bill arrives at the end of the month — long after the task, the agent that ran it,
and the reason it ran are gone.

## Why existing approaches fall short

**They look after the fact.** Tracing, LLM observability, and FinOps tooling all observe
spend once it has happened. At best they alert; none of them can stop a read while it is
executing.

**Tokens are the tip of the iceberg.** Token cost is the part that is easy to see. The
expensive part sits underneath it: query scans, data egress, and the same data read
again and again across agent steps and tasks.

## Proposal

Put cost control on the data access path, and do three things at execution time:

1. **Tag every read with an identity** — the agent, the task, and the intent behind it.
2. **Set budgets at table, partition, and snapshot level** — where the data actually lives.
3. **Degrade when a budget is exceeded** — answer from a sample, serve from cache, or
   refuse the read.

## Why it has to be the data plane

Only the data plane knows all three of: **what** was read, **how much** was read, and
**who** read it. The model layer knows who but not what; the bill knows how much but not
who or why. Enforcement needs all three in the same place at the same moment.

## Open questions

These are points the author is not sure about. Arguments against them are wanted.

1. **Is the iceberg real?** This RFC asserts that data cost dominates token cost for
   agent tasks, but has no measurements. If, for typical tasks, tokens are the larger
   share, the case for putting control in the data plane gets much weaker.
2. **Can intent be trusted?** If the agent declares its own intent, it can be wrong or
   can game the tag to get a bigger budget. If the orchestrator assigns it, the tag may
   be too coarse to be useful. Is identity (agent + task) enough without intent?
3. **Check before the read, or during it?** Pre-execution estimates from table metadata
   can be badly off; aborting mid-scan throws away work already paid for. Which one is
   the budget actually enforced on, and how wrong can the estimate be before the budget
   is meaningless?
4. **Is degradation safe?** A sampled or cached answer is a different answer. An agent
   may act on it as if it were exact. Does every degraded result need to be labelled —
   and will agents actually honour the label?
5. **Is the data plane really the only place?** Query engines already have workload
   management (resource groups, quotas), and agents holding storage credentials can read
   files without going through any enforcement point. Where the data plane can be
   bypassed, the "only this layer knows" argument does not hold.
