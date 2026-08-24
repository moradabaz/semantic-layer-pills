# When One-Line Descriptions Aren't Enough: Giving Business Logic a Home

Post 2's one-sentence column descriptions got 6/6 right — but a one-sentence description only works when a column's meaning fits in one sentence. Real business logic usually doesn't: "this counts cancellations, but per accommodation per month, and here's why a naive `created_at` comparison would be wrong, and here's which other column to use instead" is not one sentence, and cramming three of those into [`_marts.yml`](https://github.com/moradabaz/pms-semantic-layer/blob/main/agent_stack/dbt/models/marts/_marts.yml) turns a readable catalog into a wall of text nobody will read, model or human. This post is about the mechanism dbt actually provides for that problem — reusable doc blocks, plus the two governance pieces that keep them honest — and whether feeding the *deep* version of the documentation to the model closes gaps the short version left open.

## The mechanism: doc blocks, exposures, and contracts

**Doc blocks solve "where does the long explanation live without bloating every column."** A column's `description:` field in [`_marts.yml`](https://github.com/moradabaz/pms-semantic-layer/blob/main/agent_stack/dbt/models/marts/_marts.yml) can be a literal string, or a reference to a named block defined once in a separate `.md` file. `avg_booking_window_days` is the clearest real example in this project:

```yaml
# agent_stack/dbt/models/marts/_marts.yml
- name: avg_booking_window_days
  description: "{{ doc('avg_booking_window_days') }}"
```

```
# agent_stack/dbt/models/marts/_marts.md
{% docs avg_booking_window_days %}
Average number of days between when a guest booked a reservation and
their check-in date (`start_date - booked_at`)...

**Do not compute this from `reservation.created_at`.** `created_at` is a
technical row-insert timestamp, not the date the guest actually made the
booking -- for this project's seed data, most reservations were
bulk-inserted long after their real booking date, and
`start_date - created_at` comes out negative for the majority of
historical rows...
{% enddocs %}
```

[`_marts.yml`](https://github.com/moradabaz/pms-semantic-layer/blob/main/agent_stack/dbt/models/marts/_marts.yml) stays a one-liner per column; the actual paragraph lives once, in [`_marts.md`](https://github.com/moradabaz/pms-semantic-layer/blob/main/agent_stack/dbt/models/marts/_marts.md), under a name (`avg_booking_window_days`) that any column anywhere can reference. That's what makes it possible to write `mart_kpi.adr` and `mart_kpi.revpan`'s descriptions as `"{{ doc('adr') }}"` and `"{{ doc('revpan') }}"` instead of retyping the proration explanation — one paragraph, referenced twice, instead of two paragraphs that can quietly drift apart the next time someone edits one and forgets the other. The rule for *when* a column earns one of these: only when it could be misused in a way that isn't obvious from its name. Everything else stays a plain one-line string, same as post 2.

**Exposures solve "does anyone even know something real depends on this."** In [`agent_stack/dbt/models/marts/_exposures.yml`](https://github.com/moradabaz/pms-semantic-layer/blob/main/agent_stack/dbt/models/marts/_exposures.yml):

```yaml
exposures:
  - name: text_to_sql_agent
    type: application
    maturity: low
    description: >
      Answers business questions about occupancy, revenue, and
      cancellations by querying the semantic layer directly.
    depends_on:
      - ref('mart_kpi')
      - ref('mart_aggregates')
```

This is dbt metadata, not something the model ever reads. It's how the dbt project itself knows a real, named consumer (this exact agent) depends on `mart_kpi` and `mart_aggregates` — so if someone renames or drops a column those marts expose, dbt can flag it as a breaking change against *this specific consumer*, instead of the change shipping silently and the first sign of trouble being the agent giving wrong answers three weeks later.

Concretely: without this exposure, if someone later decided `mart_kpi.adr` was redundant with `revpan` and dropped it, `dbt build` would succeed — nothing in the dbt project itself references that column, so nothing would fail. The agent would keep running, keep matching the `adr_revpan` routing type, keep asking for a column that no longer exists, and every question about ADR would start failing with a raw Postgres error instead of a warning anyone saw at build time. The `depends_on: [ref('mart_kpi')]` line is what turns "some agent somewhere might use this" into "dbt itself will tell you this specific change breaks a specific, named thing" before it ships.

**Contracts solve "how do you stop the schema from silently drifting away from what the docs claim."** On `mart_aggregates`, in [`_marts.yml`](https://github.com/moradabaz/pms-semantic-layer/blob/main/agent_stack/dbt/models/marts/_marts.yml):

```yaml
models:
  - name: mart_aggregates
    config:
      contract:
        enforced: true
    columns:
      - name: available_nights
        data_type: integer
      - name: revenue
        data_type: numeric(10,2)
      # every column needs an explicit data_type once a contract is enforced
```

`enforced: true` plus an explicit `data_type:` on every column makes `dbt build` refuse to run if a column's real type stops matching what's declared here. This isn't a hypothetical — it was tested for real by deliberately editing [`mart_aggregates.sql`](https://github.com/moradabaz/pms-semantic-layer/blob/main/agent_stack/dbt/models/marts/mart_aggregates.sql) to cast `occupied_nights` to `TEXT`, with the contract still declaring `bigint`, and running `dbt run`:

```
Compilation Error in model mart_aggregates
This model has an enforced contract that failed.

| column_name     | definition_type | contract_type | mismatch_reason    |
| ---------------- | --------------- | ------------- | ------------------ |
| occupied_nights  | TEXT             | LONGINTEGER   | data type mismatch |
```

The table never got created — `dbt` stopped before running a single `CREATE TABLE`. Without the contract, that same change would have silently shipped: the model would build, the column would now be text, and the first sign of trouble would be a comparison or arithmetic operation failing somewhere downstream — in a dashboard, in a dbt test, or in a prompt an LLM never even gets to see fail cleanly, because by the time it's in a string being sent to an API, "the column changed type" and "the description is now wrong" look identical from the outside.

None of these three touch what a prompt looks like. That's the point: doc blocks are the *content*, exposures and contracts are what make trusting that content reasonable in the first place. The actual experiment below only tests the first one — whether deep, doc-block-resolved text, handed to the model, does better than post 2's one-liners.

## The experiment

Same setup as post 2 — same 4 tables, same 6 questions, no routing, no per-question filtering, same [`docs_dump.py`](https://github.com/moradabaz/pms-semantic-layer/blob/main/agent_stack/chapters/13-live-agent/agent/docs_dump.py) mechanism from that post. The only change: `CONTEXT_MODE=docs_full` instead of `docs_short`, which flips `_resolve_description`'s `short` flag to `False` — every column's full, doc-block-resolved text goes into the prompt instead of just its first sentence. [Full transcript here.](transcript.md)

## The result: 5/6 correct, 1 confidently wrong

Five questions landed exactly right. The sixth — "which month had the most cancelled reservations, and what's the lost revenue for that month" — came back with **February 2026, 2 cancellations, $62,284.22 lost.**

The real number, verified directly against the database: **January 2026, 45 cancellations, $61,605.58.**

$62,284 for 2 cancelled reservations is already implausible on its face — this dataset's nightly rates run $40–$350, not $30,000 a booking. That's worth flagging as a habit on its own: a number that's an order of magnitude off from what the rest of your data looks like deserves a second look before it goes in a slide deck.

## What actually went wrong

```sql
WITH max_cancelled_month AS (
    SELECT month, cancelled_count
    FROM public_marts.mart_aggregates
    ORDER BY cancelled_count DESC
    LIMIT 1
),
...
```

`mart_aggregates` is grained at **(accommodation_id, month)** — one row per accommodation, per month. `ORDER BY cancelled_count DESC LIMIT 1` on that raw table picks the single best *accommodation-month* (one property's worst month), not the month with the most cancellations across the whole portfolio. The query never summed across accommodations first. It's syntactically valid, it runs, and it returns something that looks exactly like the answer to the question asked — just not that answer.

> **The lesson for an Analytics Engineer:** documentation that explains *what a column means* is not the same as documentation that explains *what grain a table is at*. This project's doc blocks were full of "why" (proration logic, snapshot timing) but never one had ever been written for `cancelled_count` stating "this is per accommodation per month — sum across accommodations before ranking months." Grain is the single most common thing missing from otherwise-thorough documentation, precisely because it feels too obvious to write down until someone gets it wrong.

## What doc blocks solved relative to post 2, and what they didn't

Post 2's short descriptions happened to protect against the ADR bug because the *formula* was baked directly into the one sentence the model needed — a lucky fit between problem and format. Doc blocks solve a real, different problem: they let a real business rule (a proration caveat, a "don't use this column" warning) exist *at all* without forcing every column's description into a paragraph. That's a genuine step up from post 2 — richer, more accurate documentation, capable of holding an explanation a one-liner never could.

What they don't solve is coverage. Richness isn't the same as completeness — nobody had written a grain warning on `cancelled_count` because it hadn't been a real problem yet when the docs were written. A doc block only protects against the mistakes someone already thought to write down, no matter how good its prose is once written.

Two ways to close a gap like this, in order of preference:

1. **Fix it in the mart, not the docs.** A dedicated `mart_cancellations_by_month` (grained by month only, already summed across accommodations) removes the chance of this mistake entirely — there'd be no raw, ungrouped table for a query to misuse.
2. **If you can't add a mart today, add a doc block for the grain specifically.** Not "this counts cancellations" — "this counts cancellations *per accommodation per month*; `SUM()` this column, grouped by month, to get a portfolio-wide total."

Next post drops the full-schema dump entirely and gives the model *only* the tables relevant to each specific question — the routing approach. It doesn't automatically fix this grain problem, and it introduces a new failure mode of its own: what happens when "only the relevant tables" leaves out a table the question actually needed.
