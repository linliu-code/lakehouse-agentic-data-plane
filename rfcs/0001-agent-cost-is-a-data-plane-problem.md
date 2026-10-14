# RFC 0001: Agent cost is a data plane problem

- **Status:** Draft (v1)
- **Author:** Lin Liu
- **Created:** 2026-10-14

## Problem

A single agent task fans out into N queries and scans. Nobody sees what that cost until
the bill arrives at the end of the month — long after the task, the agent that ran it,
and the reason it ran are gone.

## Definitions

"Agentic data plane" is not an established term. This RFC uses the following meanings.

**Traditional data plane.** The layer that physically moves and changes data:
ingestion, writes and upserts, table services (compaction, clustering, cleaning), query
execution, and egress. It assumes the workload is *planned*: pipelines and queries written
by people, scheduled jobs, predictable shapes. Cost is managed after the fact, through
capacity planning and chargeback.

**Agentic data plane.** The same layer, built on the assumption that the workload is
*generated at runtime* by software that picks its next read from its last result. Three
things change:

1. Every request carries **who** is asking — agent, task, and intent — not only a
   service account.
2. Policy such as budgets and degradation is **enforced while the work executes**, not
   only metered for the bill.
3. The data plane returns **structured feedback the caller can act on** ("estimated
   2 TB; filter on `dt`"), not only results or errors.

A traditional data plane executes intent someone planned. An agentic data plane executes
intent an agent formed a moment ago, and has to govern it at runtime.

**What it is not.**

- **AI gateways** (model and MCP gateways) govern model and tool traffic: token
  metering, rate limits, routing, prompt policy. They do not see which tables,
  partitions, or bytes a tool call touched. They are the control point for the *model*
  side of agent cost — one slice of an agentic control plane, not part of the data plane.
- **The catalog** holds metadata, access control, lineage, and increasingly vends
  scoped storage credentials. It is the natural place to *define* cost policy, but it
  never sees bytes scanned, so it cannot *enforce* it.

## How agents use the data plane differently

Not yet measured; this table is the working model this RFC argues from.

| Stage | Agent behaviour | Human data engineer | Cost implication |
|---|---|---|---|
| Query | Wide fan-out: speculative queries, schema probes, retries after wrong guesses, `SELECT *` or samples "to look" | Few, deliberate queries against known tables | High variance per task, no natural upper bound. Budgets must cover a whole task, not a single query |
| Repeat reads | No memory across steps or tasks; re-derives the same intermediates | Remembers; saves intermediate tables | Caching and result reuse are the highest-leverage fix |
| Egress | Pulls rows into the model's context, possibly across regions or clouds | Reads results inside a BI tool or notebook | Bytes leave the engine. Pushing work down (aggregate before returning) is the lever |
| Writes / updates | Many small ad-hoc tables and frequent small upserts | Fewer, batched, reviewed writes | Small files and log files: the cost arrives later |
| Table services | Never runs them, but its writes trigger compaction, clustering, cleaning | Plans for them | Cost lands later and on someone else's bill; hard to trace back |
| Ingestion | Rarely moves data itself, but may *create a pipeline* | Creates pipelines after review | One agent decision becomes a recurring cost |
| Concurrency | Parallel sub-agents, unattended, around the clock | Business hours, one at a time | Nobody is watching at spike time; only runtime enforcement helps |

Two kinds of cost in this table are invisible at read time:

- **Deferred cost:** table-service work triggered by an agent's writes.
- **Recurring cost:** pipelines and materializations an agent created.

A design that only budgets reads misses both, so they are in scope here.

## Why existing approaches fall short

**They look after the fact.** Tracing, LLM observability, and FinOps tooling all observe
spend once it has happened. At best they alert; none of them can stop a read while it is
executing.

**Tokens are the tip of the iceberg.** Token cost is the part that is easy to see. The
expensive part sits underneath it: query scans, data egress, the same data read again
and again across agent steps and tasks, and the deferred and recurring cost of what
agents write.

## Proposal

Put cost control on the data access path, and do three things at execution time:

1. **Tag every read with an identity** — the agent, the task, and the intent behind it.
2. **Set budgets at table, partition, and snapshot level** — where the data actually lives.
3. **Degrade when a budget is exceeded** — answer from a sample, serve from cache, or
   refuse the read.

### Design sketch

Responsibilities split three ways: the **control plane defines**, the **data plane
enforces and measures**, and a **ledger attributes**.

1. **Identity propagation.** The orchestrator issues a signed task context (agent, task,
   intent, budget id). It travels with every tool call, into every engine session as a
   query tag, and down to storage requests. The AI gateway uses the same task id.
2. **Policy in the catalog.** Budgets are attached to data scopes (namespace → table →
   partition / snapshot) and to principals (agent / task / intent), and versioned like
   permissions.
3. **Two-stage enforcement in the data plane.**
   - *Plan time:* estimate bytes and files touched from table metadata (partition
     pruning, column stats), then admit, degrade, or deny before execution. Cheap, but
     the estimate can be wrong.
   - *Run time:* the reader counts bytes scanned and stops at the cap. This catches bad
     estimates, at the price of discarding work already done.
4. **Degradation ladder, cheapest first:** reuse a cached result → serve a materialized
   or aggregated version → sample, labelled approximate → return a structured "narrow
   your query" hint with the estimate → deny. The hint step is agent-specific: a human
   gets an alert, but an agent can read a machine-readable refusal and rewrite its
   query, so enforcement steers instead of only blocking.
5. **Per-task ledger.** Bytes scanned, files opened, egress bytes, and compute-seconds
   per task, joined by task id with token cost from the AI gateway, so one record shows
   the whole cost of a task.
6. **Attributing deferred and recurring cost.** Writes record the task id in commit
   metadata. Later table-service runs are attributed back in proportion to the files
   each task produced. Pipelines an agent creates inherit its task or owner id, so their
   recurring cost is never orphaned.

## Why it has to be the data plane

Only the data plane knows all three of: **what** was read, **how much** was read, and
**who** read it. The model layer — including AI gateways — knows who but not what; the
catalog knows what may be read but not how much was; the bill knows how much but not who
or why. Enforcement needs all three in the same place at the same moment.

## Open questions

These are points the author is not sure about. Arguments against them are wanted.

1. **Is the iceberg real?** This RFC asserts that data cost dominates token cost for
   agent tasks, but has no measurements. If, for typical tasks, tokens are the larger
   share, the case for putting control in the data plane gets much weaker.
2. **Can intent be trusted?** If the agent declares its own intent, it can be wrong or
   can game the tag to get a bigger budget. If the orchestrator assigns it, the tag may
   be too coarse to be useful. Is identity (agent + task) enough without intent?
3. **Is plan-time estimation accurate enough to enforce on?** If byte estimates from
   table metadata are often badly off, enforcement falls back to run-time aborts, which
   throw away paid-for work. How wrong can the estimate be before plan-time budgets are
   meaningless?
4. **Is degradation safe?** A sampled or cached answer is a different answer. An agent
   may act on it as if it were exact. Does every degraded result need to be labelled —
   and will agents actually honour the label?
5. **Can the data plane be bypassed?** Query engines already have workload management
   (resource groups, quotas), and agents holding raw storage credentials can read files
   without passing any enforcement point. Catalog credential vending limits *what* an
   agent can read, but not *how much*; metering that still needs an engine, reader, or
   storage proxy in the path. Where none of those is in the path, the "only this layer
   knows" argument does not hold.
