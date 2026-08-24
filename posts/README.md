# Publication plan: "How Much Context Does a Model Need to Answer SQL Questions?"

> Kept in this repo, separate from [pms-semantic-layer](https://github.com/moradabaz/pms-semantic-layer), so drafts of posts not yet published don't sit in the same place as the public code. All code/dbt/agent paths referenced below link back to that repo.

A Medium series + YouTube companion, in English, built from this project's real artifacts. Every log line, number, and diagram in these posts comes from something we actually ran against our own PMS database. If we don't have the real artifact yet, the post doesn't get written yet.

## The series' actual question

Chapter 07's experiment (post 1) showed an LLM with a database and zero context fails completely. The obvious next question is "so give it context" — but that's not one decision, it's several, made incrementally: raw schema, documentation, business-logic notes, per-question routing, real access control. Each of those is a real, separate experiment in this series, run against the exact same 6 business questions, so the 5 posts form one direct, comparable progression instead of 5 unrelated case studies.

## Working principles

- **Hook first.** Every post opens with a failure or a surprising result, not a solution stated upfront.
- **Real artifacts only.** No fabricated logs, no invented error messages, no staged "before" state. Every intermediate context level in this series was actually run, including reconstructing this project's own pre-fix code state from git history to get a real "before" transcript, not a rewritten one.
- **Every number gets checked.** A query running without an error is not evidence it's correct — every result quoted in these posts was independently verified against the database directly.
- **Written after the experiment, not before.** A post's `post.md` gets written once its underlying transcript exists and has been reviewed.
- **Medium formatting**: short paragraphs, copyable code blocks, callout-style `> **Key lesson:**` blockquotes, comparison tables where they clarify more than prose would.

## Series structure

| # | Post title | Context given to the model | Real finding |
|---|---|---|---|
| 1 | "What Happens When You Give an LLM a Database and Zero Context" | Nothing beyond a generic system prompt | 6/6 wrong: 5 hallucinated tables, 1 fabricated number with no query run at all |
| 2 | "Can Column Descriptions Alone Stop an LLM From Hallucinating SQL?" | Full, unfiltered semantic layer schema + one-line column descriptions, no routing | 6/6 correct — a formula stated in a one-line description prevented a bug that resurfaces in post 5 |
| 3 | "When One-Line Descriptions Aren't Enough: Giving Business Logic a Home" | Same full schema + doc blocks with the deep "why", plus exposures/contracts as the governance behind them | 5/6 correct — a grain mismatch (per-accommodation-month vs. portfolio-wide) produced a confident, wrong answer that richer documentation didn't catch |
| 4 | "Routing: Giving the Model Only What the Question Needs" | Routing: only the 1-2 tables relevant to the matched question, real config, pre-fix state | Mixed: a caught typo, an honest fallback, an honest refusal, and the same grain bug — different failure shapes than posts 2-3, not fewer failures |
| 5 | "Locking It Down: Real Access Control on Top of Good Context" | Fixed routing + a real, verified read-only database role + a hard row cap | 6/6 verified correct on the original questions; a second round of 6 new questions found 4 more real bugs immediately, including a model silently answering a different question than the one asked |

## Bonus / deferred

- **"9 Real Errors and How We Fixed Them"** — a standalone catalog of every real bug found while building the dbt semantic layer itself (staging-layer mistakes, proration bugs, a data generator that violated its own invariant). Drafted, held back from the main 5-post arc so it doesn't duplicate the bugs already covered inside posts 2-5 -- may run as a bonus post or get folded into a wrap-up.
- **"From YAML to a Live Agent"** — the engineering-focused walkthrough of chapter 13's architecture itself (routing.py, the Postgres role, fetchmany enforcement), for a more technical/builder audience than the 5-post narrative arc.

## Status

Posts 1 and 5 are built directly from this project's `agent_stack/chapters/07` and `agent_stack/chapters/13` artifacts. Posts 2, 3, and 4 required building new experiments specifically for this series ([`agent_stack/chapters/13-live-agent/agent/main_experiment.py`](https://github.com/moradabaz/pms-semantic-layer/blob/main/agent_stack/chapters/13-live-agent/agent/main_experiment.py), [`docs_dump.py`](https://github.com/moradabaz/pms-semantic-layer/blob/main/agent_stack/chapters/13-live-agent/agent/docs_dump.py)) and, for post 4, temporarily reconstructing this project's own pre-fix code state from git history to capture a real "before" transcript rather than reusing notes from memory. All 5 are written and reviewed.
