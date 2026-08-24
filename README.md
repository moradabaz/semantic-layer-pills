# Semantic Layer Pills

Posts, pills, and interview prep built while working on
[pms-semantic-layer](https://github.com/moradabaz/pms-semantic-layer) — a
project about giving an LLM enough real context (dbt semantic layer,
documentation, routing, access control) to answer business questions in
SQL without hallucinating.

This repository is deliberately separate from the source project: that one
holds only the code, the dbt models, and the tooling. This one holds
everything meant for readers — the post series (including drafts not yet
published), the shorter "pills" distilled from it, and the study track
built from a senior review of the series.

## Contents

- [`posts/`](posts/) — the "How Much Context Does a Model Need to Answer SQL Questions?" series. See [`posts/README.md`](posts/README.md) for the plan and status.
- [`interview-prep/01-dbt-semantic-layer-governance.md`](interview-prep/01-dbt-semantic-layer-governance.md) — doc blocks, schema separation, exposures, contracts, source freshness.
- [`interview-prep/02-agent-routing-and-scope.md`](interview-prep/02-agent-routing-and-scope.md) — routing vs. scope, writing a `question_types` entry, fallback design, access-control collisions.
- [`interview-prep/03-live-agent.md`](interview-prep/03-live-agent.md) — enforcing scope at the database layer, real bugs found by running the agent, robustness relative to the questions actually tried.
- [`interview-prep/04-senior-review-walkthrough.md`](interview-prep/04-senior-review-walkthrough.md) — the full question-by-question Socratic walkthrough for the senior review of the post series, plus the cross-cutting patterns that repeat across all of it.

All code, dbt models, and agent configs referenced throughout these
documents live in
[github.com/moradabaz/pms-semantic-layer](https://github.com/moradabaz/pms-semantic-layer) —
links are inlined wherever a specific file is mentioned.
