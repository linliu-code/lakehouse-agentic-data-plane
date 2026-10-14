# Changelog

All notable changes to this repository are documented here.
The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Changed
- RFC 0001 rewritten to the v1 skeleton: problem, why existing approaches fall short,
  proposal (identity tags, table/partition/snapshot budgets, degradation), why the data
  plane, and five open questions.
- README thesis updated to match RFC 0001 v1.
- RFC 0001: added Definitions (agentic vs. traditional data plane; AI gateway and
  catalog roles), an agent-vs-human behaviour table, deferred and recurring cost in
  scope, and a design sketch (identity propagation, policy in catalog, two-stage
  enforcement, degradation ladder with "narrow your query" feedback, per-task ledger,
  attribution of deferred cost). Open questions 3 and 5 sharpened.

### Added
- README with the one-sentence thesis.
- RFC 0001: Agent cost is a data plane problem (Draft).
- This changelog.
