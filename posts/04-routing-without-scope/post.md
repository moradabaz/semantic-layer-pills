# Routing: Giving the Model Only What the Question Needs

Posts 2 and 3 dumped the entire semantic layer into every prompt. That doesn't scale past a handful of tables — which is the actual reason routing exists: match the question to a small, relevant slice of the schema, and hand over only that. This post runs the routing mechanism from this project's chapter 12, in the state it was actually in before two real bugs in it got found and fixed. Less noise, but not automatically fewer mistakes.

## The experiment

A routing config maps keyword patterns to a `question_type`: which table(s) answer this kind of question, how to join them, and any business-logic caveats specific to that question. The model only sees the 1-2 tables its matched type points to — everything else in the schema is invisible to it. It's a plain YAML file, [`agent_stack/agent_config/agent_routing.yml`](https://github.com/moradabaz/pms-semantic-layer/blob/main/agent_stack/agent_config/agent_routing.yml), not a framework or a library. [Full transcript here.](transcript.md)

**How one `question_type` entry actually gets written.** Every entry follows the same checklist: `type` (a stable identifier for the *shape* of the question, not the literal wording), `keywords` (real phrasings a user might type, including synonyms), `primary_mart` (the most-derived table that already answers most of the question), `secondary_tables`/`join_keys` (only when the primary mart's grain is missing something), and `notes` (the one non-obvious caveat nothing else in the config can express). This is the real entry for "revenue by property manager," unedited:

```yaml
# agent_stack/agent_config/agent_routing.yml
- type: revenue_per_property_manager
  keywords: ["property manager", "manager's portfolio", "each of their accommodations"]
  primary_mart: mart_kpi
  columns: [revpan, adr]
  secondary_tables: [dim_accommodation]
  join_keys:
    - "mart_kpi.accommodation_id = dim_accommodation.accommodation_id"
  notes: >
    dim_accommodation.property_manager_id is the CURRENT manager.
    fct_reservation deliberately excludes property_manager_id (it's a
    frozen snapshot at booking time) -- a historical variant of this
    question should hit the fallback, not reach into a different table.
```

The `notes` field is where the real decision lives: this entry joins `dim_accommodation` (the accommodation's *current* manager), not `fct_reservation` (a frozen snapshot of whoever managed it *at booking time*) — because those answer two different questions, and only a person deciding the routing config, not a matching algorithm, knows which one a given question actually means. Two more blocks sit above all the `question_types` entries in the same file:

```yaml
fallback:
  action: ask_clarification
  never: guess_or_query_without_a_matched_type
  message_template: >
    I don't have a defined routing rule for this question. Can you
    rephrase it...

matching_rules:
  on_multiple_matches: prefer_the_more_specific_type
```

Both were written before a single real question was tested against the config — and both are about to get exercised for real below.

## How the config actually gets read

A YAML file doesn't match anything by itself — a function does, and it's worth seeing exactly what this one does, because its limitations are the whole story of this post. This is the real, unedited `match_question_type` as it existed for this experiment (fixed later, in post 5):

```python
# agent_stack/chapters/13-live-agent/agent/routing.py

def match_question_type(question, routing_config):
    question_lower = question.lower()
    best_type = None
    best_keyword_len = -1

    for question_type in routing_config["question_types"]:
        for keyword in question_type["keywords"]:
            if keyword.lower() in question_lower and len(keyword) > best_keyword_len:
                best_type = question_type
                best_keyword_len = len(keyword)

    return best_type
```

This is a substring search, nothing more: for every keyword of every `question_type`, check if that exact phrase appears anywhere in the lowercased question; the longest matching keyword wins. Run it by hand on this post's Q4 — "Which accommodations have an ADR lower than their **normalized daily cost**..." — against `adr_vs_cost`'s keyword `"normalized cost"`: is the literal substring `"normalized cost"` present in `"normalized daily cost"`? No — the word "daily" sits in between, so `keyword.lower() in question_lower` is `False`, and that keyword never even enters the comparison. That's the entire mechanism behind Q4's misrouting below, before a single query gets generated.

Whichever `question_type` wins, a second function turns it into the actual prompt:

```python
# agent_stack/chapters/13-live-agent/agent/schema_context.py

def build_context(question_type, columns_by_table):
    lines = [f"Question type: {question_type['type']}", ""]
    tables = [question_type["primary_mart"], *question_type.get("secondary_tables", [])]
    for table in tables:
        lines.append(f"Table: public_marts.{table}")
        for column_name, data_type in columns_by_table.get(table, []):
            lines.append(f"  - {column_name} ({data_type})")
        lines.append("")
    if question_type.get("join_keys"):
        lines.append("Join keys:")
        for join_key in question_type["join_keys"]:
            lines.append(f"  - {join_key}")
    lines.append(f"Notes: {question_type['notes'].strip()}")
    return "\n".join(lines)
```

`columns_by_table` itself comes from a live `information_schema` query (`get_live_columns`, same file) — the same "don't trust the YAML to be in sync, check Postgres directly" principle from post 2's [`docs_dump.py`](https://github.com/moradabaz/pms-semantic-layer/blob/main/agent_stack/chapters/13-live-agent/agent/docs_dump.py). Put together: `match_question_type` picks one `question_type` out of the whole config; `build_context` renders *only* that type's `primary_mart` + `secondary_tables`, with real, live column names and types, plus its `join_keys` and `notes`. Everything else in the schema — every other table in this project — is simply never written into the string handed to the model.

The command that runs all of this end to end, using this exact pre-fix routing state (recovered from this project's own git history for this post, then restored afterward):

```bash
docker compose run --rm \
  -e TRANSCRIPT_PATH=/content/04-routing-without-scope/transcript.md \
  live-agent python main.py
```

[`main.py`](https://github.com/moradabaz/pms-semantic-layer/blob/main/agent_stack/chapters/13-live-agent/agent/main.py) loops over the 6 questions, calling `match_question_type` then `build_context` then `ask_model` for each one — the same `ask_model`/`messages` mechanism from post 2, just with a much smaller `context` string plugged in. This run uses the routing config as it stood right after being built, before it had been exercised against real questions.

## The result: a mixed bag, and every failure is a different shape

**Q1 — a typo, caught immediately.** The model wrote `SUM(k.revpand)` — one extra letter — and Postgres rejected it outright: `column k.revpand does not exist`. Visible, loud, easy to fix. This is the *good* kind of failure: wrong, but impossible to mistake for a real answer.

**Q2 — an honest "I don't know."** "How many cancelled reservations has accommodation 15 had" matched no `question_type` at all. The system's designed fallback kicked in: *"I don't have a defined routing rule for this question. Can you rephrase it...?"* No guess, no query. This is routing doing exactly its job — the gap here is coverage (nobody had anticipated this exact phrasing), not correctness.

**Q4 — a different kind of honesty.** The matched type only exposed `mart_kpi` — no cost data anywhere in scope. The model didn't invent a cost column. It replied: *"the only table I have access to does not contain any column that represents a normalized daily cost... I can't compare it."* Compare this to post 2, where the same question got a correct, silent answer, because that experiment's full-schema context happened to include the cost table this routed context excluded. Narrower context traded a right answer for an honest refusal — not a bad trade, but not a free one either.

**Q5 — the same grain bug from post 3, worse.** The question asked for *the* month with the most cancellations. The query returned **15 rows**, several of them duplicates:

```
(2026-02-01, 2, $62,284.22)
(2026-01-01, 2, $61,605.58)
(2026-07-01, 2, $53,587.96)
(2026-02-01, 2, $62,284.22)   <- duplicate
...
```

The model selected `month, cancelled_count` straight from `mart_aggregates` — same (accommodation, month) grain problem as post 3 — and then filtered for `cancelled_count = MAX(cancelled_count)` without ever grouping across accommodations. Since a "2 cancellations in one month" accommodation-month is common, the filter matched fifteen of them at once. This isn't a new bug; it's proof that neither full-context documentation (post 3) nor scoped routing, on its own, closes a grain-mismatch gap that nobody wrote down explicitly.

## What routing changed, and what it didn't

Routing's real job is reducing noise — fewer tables in view means fewer chances to reference something irrelevant or nonexistent. It did that here: no hallucinated table names anywhere in this run, a real improvement over the zero-context baseline from post 1. What it does *not* do automatically:

- **It doesn't fix a grain bug.** A grain mismatch lives in one table's own definition, and no amount of trimming the context down to "just the right tables" removes it if the right table is still ambiguous about its own grain.
- **It can accidentally remove a table the question actually needs**, trading a wrong answer for an honest refusal (Q4) — better than being wrong, but still not the goal.
- **It says nothing yet about what the agent is *allowed* to do.** Every query above still ran through the same `is_read_only` check and a role with no write access — but nothing here stops it from reading a schema it shouldn't, or a column that shouldn't be in an analytics agent's hands at all (a guest's name, a manager's email). That's a different guarantee, from a different layer, and it's what closes out this series.

> **The lesson for an Analytics Engineer:** a routing config is a map of *where to look*, not a guarantee of *what's correct once you get there*. Test it against real questions before trusting it — this project found both of these gaps (a missing keyword, a missing table) by actually running the agent, not by reading the YAML.

Two fixes closed the gaps found here. Q2's fallback was closed with a routing keyword covering that exact phrasing. Q4's misrouting had a different cause worth naming precisely: the question said "normalized **daily** cost," the correct type's keyword was "normalized cost" — and exact-phrase matching doesn't see past the inserted word, so the question fell through to the wrong, cost-blind type instead. The fix was switching from phrase matching to matching on the *set of words* in a keyword, regardless of their order or what's inserted between them. Neither fix was a smarter algorithm searching for gaps on its own — both were a person looking at one specific real failure and deciding exactly what the config was missing.
