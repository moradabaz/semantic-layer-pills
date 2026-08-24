# Can Column Descriptions Alone Stop an LLM From Hallucinating SQL?

Chapter one of this series ended with an LLM given a database and nothing else: 6 questions, 6 wrong answers, including a fabricated "70% occupancy rate" with no query behind it at all. The fix seems obvious — give it the schema. This post tests exactly that, and only that: no routing, no per-question filtering, just the entire semantic layer dumped into the prompt every single time. Before running that experiment, this post shows exactly where "the schema" comes from and how it physically gets into a prompt — none of it happens by magic.

## Where the documentation actually lives

Nothing in post 1 had any documentation at all — the model queried the raw application database directly. Everything from here on runs against a **dbt project** built specifically to sit between the raw Postgres tables and anything that reads from them (a BI tool, an analyst, or an LLM). Its layout:

```
agent_stack/dbt/models/
├── staging/
│   ├── _sources.yml          # documents the 8 raw tables (1 per source table)
│   ├── _staging.yml          # tests + descriptions for the staging models below
│   ├── stg_accommodation.sql
│   ├── stg_reservation.sql
│   └── ... (6 more, one per raw table, 1:1 passthroughs)
└── marts/
    ├── _marts.yml            # descriptions + tests for every mart column
    ├── _marts.md             # long-form {% docs %} blocks, referenced from _marts.yml
    ├── _exposures.yml        # declares the agent as a real downstream consumer
    ├── mart_kpi.sql           # 6 business KPIs (occupancy, ADR, RevPAN, ...)
    ├── mart_aggregates.sql    # monthly revenue/occupancy per accommodation
    ├── fct_reservation.sql    # one row per reservation, PII stripped
    └── dim_accommodation.sql  # one row per accommodation
```

`staging/` is a boring, 1:1 mirror of the raw tables — no joins, no business rules, just renamed/typed columns. `marts/` is where the actual business logic lives (KPI formulas, proration, PII removal) — and where [`_marts.yml`](https://github.com/moradabaz/pms-semantic-layer/blob/main/agent_stack/dbt/models/marts/_marts.yml) puts a `description:` on every single column. This is the real file, unedited, for the `adr` column on `mart_kpi`:

```yaml
# agent_stack/dbt/models/marts/_marts.yml
      - name: adr
        description: "{{ doc('adr') }}"
```

`{{ doc('adr') }}` is a reference into [`_marts.md`](https://github.com/moradabaz/pms-semantic-layer/blob/main/agent_stack/dbt/models/marts/_marts.md), where the actual text lives:

```
# agent_stack/dbt/models/marts/_marts.md
{% docs adr %}
Average Daily Rate: revenue / occupied_nights. Both inputs are prorated
across month boundaries -- see mart_aggregates' revenue/occupied_nights
docs. NULL (not 0) when occupied_nights is 0, meaning there's nothing to
average, not a $0 rate.
{% enddocs %}
```

This is standard dbt — `dbt docs generate` reads exactly this and builds a browsable documentation site out of it. None of it is aimed at an LLM. That's the next problem: getting text that lives in a `.yml`/`.md` pair, meant for `dbt docs generate`, into the string that actually becomes a model's prompt.

## How a YAML description becomes a prompt

Two real, small functions bridge the gap, and it's worth seeing both — this is the part a "give it the schema" one-liner always skips.

**Step 1 — turn the dbt files into a plain-text block.** [`docs_dump.py`](https://github.com/moradabaz/pms-semantic-layer/blob/main/agent_stack/chapters/13-live-agent/agent/docs_dump.py), in the agent's own codebase, does exactly what `dbt docs generate` does (read [`_marts.yml`](https://github.com/moradabaz/pms-semantic-layer/blob/main/agent_stack/dbt/models/marts/_marts.yml), resolve `{{ doc(...) }}` refs against [`_marts.md`](https://github.com/moradabaz/pms-semantic-layer/blob/main/agent_stack/dbt/models/marts/_marts.md)), except instead of building an HTML page it builds a string:

```python
# agent_stack/chapters/13-live-agent/agent/docs_dump.py

def _resolve_description(raw, doc_blocks, short):
    match = DOC_REF_RE.fullmatch(raw.strip())
    text = doc_blocks.get(match.group(1), raw) if match else raw
    text = " ".join(text.split())
    if short:
        return text.split(". ")[0].rstrip(".") + "."   # first sentence only
    return text

def build_full_schema_context(conn, marts_yml_path, marts_md_path, short):
    doc_blocks = _load_doc_blocks(marts_md_path)          # parses _marts.md
    marts_yml = yaml.safe_load(open(marts_yml_path))       # parses _marts.yml

    # live, not cached -- so the prompt reflects the REAL schema right
    # now, not whatever _marts.yml said the last time someone edited it
    cur.execute("""
        SELECT table_name, column_name, data_type
        FROM information_schema.columns
        WHERE table_schema = 'public_marts' AND table_name = ANY(%s)
    """, (MART_TABLES,))

    lines = []
    for table in MART_TABLES:
        lines.append(f"Table: public_marts.{table}")
        for column_name, data_type in columns_by_table[table]:
            desc = _resolve_description(column_docs.get(column_name), doc_blocks, short)
            lines.append(f"  - {column_name} ({data_type}) -- {desc}")
    return "\n".join(lines)
```

This function's return value is exactly the text block shown further down in this post ("Table: public_marts.mart_kpi..."). `short=True` is the only thing this post's experiment sets — it's the one line, `text.split(". ")[0]`, that turns [`_marts.md`](https://github.com/moradabaz/pms-semantic-layer/blob/main/agent_stack/dbt/models/marts/_marts.md)'s full paragraph into a single sentence. Concretely, for `avg_booking_window_days`, this is what that one line does to the real text:

```
Full doc block (post 3 uses this):
  "Average number of days between when a guest booked a reservation and
  their check-in date (start_date - booked_at)... Do not compute this
  from reservation.created_at. created_at is a technical row-insert
  timestamp, not the date the guest actually made the booking..."

After short=True (this post uses this):
  "Average number of days between when a guest booked a reservation and
  their check-in date (start_date - booked_at)."
```

The warning about `created_at` — a real bug this project hit and fixed — is gone. It happens not to matter for any of this post's 6 questions, so the short version still gets 6/6. It's exactly the kind of thing a short description can silently drop, which is the reason post 3 exists: to test what happens once that warning is back in the prompt.

**Step 2 — that string becomes the system prompt, and the system prompt becomes an actual API call.** [`llm.py`](https://github.com/moradabaz/pms-semantic-layer/blob/main/agent_stack/chapters/13-live-agent/agent/llm.py), unchanged:

```python
# agent_stack/chapters/13-live-agent/agent/llm.py

def build_system_prompt(context: str) -> str:
    return (
        "You are a helpful PostgreSQL assistant for a property management "
        "system database. Only query the tables described below -- they "
        "are the only ones you have access to. Write a single read-only "
        "SQL query (SELECT or WITH) that answers the question.\n\n"
        f"{context}"
    )

def ask_model(question: str, context: str) -> str:
    messages = [
        {"role": "system", "content": build_system_prompt(context)},
        {"role": "user", "content": question},
    ]
    response = client.chat.completions.create(model=OPENAI_MODEL, messages=messages)
    return response.choices[0].message.content or "Could not generate an answer."
```

That `messages` list is the literal thing sent over the wire to the model — `context` (the string `build_full_schema_context` returned) sitting inside the `system` message, and the plain business question as the `user` message. There's no other channel. If a column's description isn't in that string, the model has never seen it, full stop — and if you're wondering whether your own agent is "giving the model enough context," this is the exact place to go look: not a config file, not an intention, the actual `messages` list your code builds right before the API call.

**Step 3 — the actual command that runs all of this.** Everything above is wired together by one small orchestration script, [`main_experiment.py`](https://github.com/moradabaz/pms-semantic-layer/blob/main/agent_stack/chapters/13-live-agent/agent/main_experiment.py), and one command:

```bash
docker compose run --rm \
  -e CONTEXT_MODE=docs_short \
  -e TRANSCRIPT_PATH=/content/02-schema-and-docs/transcript.md \
  live-agent python main_experiment.py
```

[`main_experiment.py`](https://github.com/moradabaz/pms-semantic-layer/blob/main/agent_stack/chapters/13-live-agent/agent/main_experiment.py) does nothing clever — for each of the 6 questions, it calls `build_full_schema_context(conn, ..., short=True)` **once** (the same context string for every question, since this post's whole point is *no* per-question filtering), then `ask_model(question, context)`, then writes the raw response and the extracted SQL's execution result to `TRANSCRIPT_PATH`. `CONTEXT_MODE=docs_short` is the one environment variable that decides `short=True` vs. `short=False` — flip it to `docs_full` and you get post 3's experiment instead, with the exact same command and the exact same code. That's the whole reproduction path: this dbt project's [`_marts.yml`](https://github.com/moradabaz/pms-semantic-layer/blob/main/agent_stack/dbt/models/marts/_marts.yml)/[`_marts.md`](https://github.com/moradabaz/pms-semantic-layer/blob/main/agent_stack/dbt/models/marts/_marts.md), this one script, this one flag.

## The experiment

Same 6 business questions as post 1. For every question, the model gets the full `public_marts` schema — all 4 marts, every column, every type, plus that one-sentence description — built by the mechanism above, run once per question. No notes about *why* a column exists beyond that one sentence, no hint about which table answers which question, no routing. [Full transcript here.](transcript.md) This is the actual text the model received:

```
Table: public_marts.mart_kpi
  - accommodation_id (bigint) -- The accommodation this row rolls up.
  - month (date) -- First day of the calendar month this row covers.
  - occupancy_rate_pct (numeric) -- 100 * occupied_nights / available_nights.
  - adr (numeric) -- Average Daily Rate: revenue / occupied_nights.
  - revpan (numeric) -- Revenue Per Available Night: revenue / available_nights.
  ...
Table: public_marts.mart_aggregates
  ...
Table: public_marts.fct_reservation
  ...
Table: public_marts.dim_accommodation
  ...
```
(Every column got a description like this — the `...` above is just this post trimming the example, not a gap in what the model actually saw.)

## The result: 6/6 correct, verified

Every one of the 6 questions came back right — cross-checked, not taken on faith. That includes the ADR-vs-cost question, which is worth pausing on, because a later post in this series shows the model getting this exact question **wrong** under a different setup.

## Why this one worked

Look at the description above for `adr`: *"Average Daily Rate: revenue / occupied_nights."* One sentence, and it states the formula — because that's literally the text from [`_marts.md`](https://github.com/moradabaz/pms-semantic-layer/blob/main/agent_stack/dbt/models/marts/_marts.md), unmodified. When the model needed ADR for the last 3 months, it didn't invent a ratio — its query used `AVG(mk.adr)` (a window function over `mart_kpi`, aliased `mk`) directly, because the column it needed was sitting right there with an unambiguous definition attached.

> **The lesson for an Analytics Engineer:** a one-sentence column description that states the *formula*, not just the name, closes off an entire category of error before it can happen. "Average Daily Rate" as a description tells a reader what the column is called. "revenue / occupied_nights" tells them — and a model — how to compute it correctly from scratch if they ever have to. Write the second one, in the dbt project itself, and any tool that reads that documentation gets the benefit — not just this experiment.

## What this experiment does *not* prove

This result is easy to over-read, so three things worth being precise about:

1. **It worked because the schema is tiny.** Four tables, 37 columns total. Dumping an entire schema into a prompt does not scale to a production warehouse with hundreds of tables — you'd blow past context limits and drown the model in irrelevant noise long before you got to four tables. This experiment isolates *whether documentation alone helps*, not *whether "just send everything" is a viable strategy at scale.
2. **Nothing here validates against a data contract.** The model saw column types as strings in a prompt; nothing checked that `adr` is still `numeric` by the time the query runs. That's a different kind of guarantee, and it doesn't come from a good prompt.
3. **Read-only enforcement is still doing real work underneath this.** Every query above ran through the same `is_read_only` check from post 1's demo, against a dedicated Postgres role (`pms_agent`) that literally cannot write anywhere — the same role built for post 5's access-control work, reused here because it already existed by the time this experiment ran. Good context reduces *wrong* answers; it does nothing about a malicious or careless one, which is a separate problem with a separate fix, covered in full in post 5.

Next post keeps the same full, unfiltered schema dump and the same [`docs_dump.py`](https://github.com/moradabaz/pms-semantic-layer/blob/main/agent_stack/chapters/13-live-agent/agent/docs_dump.py) mechanism, but flips `short=False` — swapping the one-line descriptions for the deeper "why" documentation this project accumulated over several chapters — and finds a bug that this post's approach didn't catch.
