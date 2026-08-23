# Semantic Layer Pills

Study material and interview prep built while working on
[pms-semantic-layer](https://github.com/moradabaz/pms-semantic-layer) — a
project about giving an LLM enough real context (dbt semantic layer,
documentation, routing, access control) to answer business questions in
SQL without hallucinating.

This repository is deliberately separate from the source project: that one
holds the code, the dbt models, and the blog post series being built there.
This one holds the study track — the questions a Staff Data Engineer asked
after reviewing the series, and the walkthroughs built to actually
understand each answer, not just memorize it.

## Contents

- [`interview-prep/01-dbt-semantic-layer-governance.md`](interview-prep/01-dbt-semantic-layer-governance.md) — doc blocks, schema separation, exposures, contracts, source freshness.
- [`interview-prep/02-agent-routing-and-scope.md`](interview-prep/02-agent-routing-and-scope.md) — routing vs. scope, writing a `question_types` entry, fallback design, access-control collisions.
- [`interview-prep/03-live-agent.md`](interview-prep/03-live-agent.md) — enforcing scope at the database layer, real bugs found by running the agent, robustness relative to the questions actually tried.
- [`interview-prep/04-senior-review-walkthrough.md`](interview-prep/04-senior-review-walkthrough.md) — the full question-by-question Socratic walkthrough for the senior review of the blog post series, plus the cross-cutting patterns that repeat across all of it.

All code, dbt models, agent configs, and blog posts referenced throughout
these documents live in
[github.com/moradabaz/pms-semantic-layer](https://github.com/moradabaz/pms-semantic-layer) —
links are inlined wherever a specific file is mentioned.
