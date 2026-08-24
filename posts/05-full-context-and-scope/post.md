# Locking It Down: Real Access Control on Top of Good Context

Four posts, four experiments: no context, full documentation, deeper documentation, scoped-but-unguarded routing. Each one traded a set of failures for a different set. This post adds the last missing piece — real access control, not just good context — runs the fixed routing config against chapter 07's original 6 questions, and then pushes past them with 6 new ones designed to test how robust the setup actually is. The first run is clean. The second finds four more real bugs in the first six questions asked.

## What's new here

Everything from post 4 (routing: match a question, hand over only the relevant tables), plus:

- **A dedicated, read-only database role.** Not an app-level check saying "please don't write" — a Postgres role with `SELECT` granted on the semantic layer's schema and nothing else, verified by actually trying to break it: reading the restricted schema fails with `permission denied`, writing anywhere fails the same way, regardless of how the SQL is phrased.
- **A hard cap on rows returned**, enforced at the database-driver level (`fetchmany(500)`), independent of whether the generated SQL remembered to `LIMIT` anything.
- **The two fixes from post 4**: word-based keyword matching, and the missing keyword for "cancelled reservations."

**Where the policy is written down.** Before touching Postgres, the actual rules live in a plain YAML file, [`agent_stack/agent_config/agent_scope.yml`](https://github.com/moradabaz/pms-semantic-layer/blob/main/agent_stack/agent_config/agent_scope.yml) — the equivalent of [`agent_routing.yml`](https://github.com/moradabaz/pms-semantic-layer/blob/main/agent_stack/agent_config/agent_routing.yml), but for "what's this agent allowed to touch" instead of "which table answers this." Trimmed to its structure:

```yaml
max_rows:
  limit: 500
  reason: "no question_type legitimately needs more than 613 rows..."

read_only:
  enforced: true
  forbidden_statement_types: [INSERT, UPDATE, DELETE, DROP, ALTER, ...]

allowed_schemas: [public_marts]

forbidden_schemas:
  - schema: public_staging
    reason: "Uncurated 1:1 mirror -- no business-rule guarantees."
  - schema: public
    reason: "The operational schema the app writes to directly."

forbidden_columns:
  columns:
    - column: guest_name
      reason: "Free-text guest name -- direct PII."
    - column: email
      reason: "Direct PII."
    # ... phone, first_name, last_name
```

Writing this file down first, before writing any SQL, is the point — it's a design decision made deliberately (what's allowed, and *why*), not a set of permissions reverse-engineered from whatever happened to be convenient. Turning `allowed_schemas`/`forbidden_schemas` into something Postgres actually enforces is the next step:

**Building the database role, step by step.** This is the part that's easy to skip and shouldn't be:

```sql
CREATE ROLE pms_agent WITH LOGIN PASSWORD '...';

GRANT USAGE ON SCHEMA public_marts TO pms_agent;
GRANT SELECT ON ALL TABLES IN SCHEMA public_marts TO pms_agent;

-- so tables added to public_marts by future dbt runs stay covered,
-- without re-running this grant by hand every time
ALTER DEFAULT PRIVILEGES IN SCHEMA public_marts
    GRANT SELECT ON TABLES TO pms_agent;

-- explicit, not just "absence of a grant" -- documents the intent
REVOKE ALL ON SCHEMA public, public_staging FROM pms_agent;
```

Then it got verified, not assumed — three real connection attempts, not a reading of the GRANT statements: `SELECT` against `public_marts` succeeded; the exact same query against `public_staging` failed with `permission denied for schema public_staging`; an `INSERT` against a `public_marts` table failed with `permission denied for table`. If you're setting this up yourself: write the role, then try to break it in all three directions before trusting it.

**Why `fetchmany(500)` and not some other number.** The cap has to be grounded in something real, not picked round. This project's own `dim_accommodation` has 613 rows and `fct_reservation` has ~5,300 — no question this routing config answers legitimately needs more rows than that, and 500 sits comfortably below the largest table's full count while leaving room for any reasonable `GROUP BY` result. And this cap is enforced by how many rows the Python driver pulls off the cursor (`cur.fetchmany(500)`), not by trusting the model to remember a `LIMIT` clause — the same principle as chapter 07's own `fetchmany(6)`, just sized for a real analytics agent instead of a demo transcript.

## The result: 6/6, verified against the database

[Full transcript here.](https://github.com/moradabaz/pms-semantic-layer/blob/main/agent_stack/chapters/13-live-agent/transcript.md) Every one of the 6 questions matched a routing type and returned a result that checked out against a direct manual query — the same rigor applied to every number in this series, not a pass because the SQL happened to run.

That clean result is the end state. It is not the first state. Building this routing config surfaced two real, confidently-wrong answers along the way — and one of them is worth reproducing here, exactly, because "a bug happened and we fixed it" is a weaker claim than showing the actual query.

## Bug: ADR computed with RevPAN's formula

Before `mart_kpi` was added to this question's routing entry, the model only saw `mart_aggregates` (raw `revenue`, `available_nights`, `occupied_nights`) and `dim_accommodation` (cost) — not the table that already had a correct, tested `adr` column. Left to reconstruct the formula itself, this is what it wrote ([full transcript](transcript-adr-bug-before-fix.md)):

```sql
adr_last_3_months AS (
    SELECT accommodation_id,
           SUM(revenue) / SUM(available_nights) AS adr
    FROM public_marts.mart_aggregates
    WHERE month >= date_trunc('month', current_date - INTERVAL '3 months')
      AND month <  date_trunc('month', current_date)
    GROUP BY accommodation_id
)
```

`revenue / available_nights` is RevPAN's formula. ADR is `revenue / occupied_nights`. The query ran cleanly and returned a real number for every accommodation — nearly all zero, because dividing by the larger denominator (available nights instead of occupied nights) crushed the result toward nothing for anyone with low occupancy. Nothing errored. Nothing looked obviously broken. This is chapter 07's original thesis, one layer up: a real, scoped, non-hallucinated context still produced a confidently wrong number, because the one table with the right answer already in it wasn't in view.

The fix was adding `mart_kpi` to that routing entry's table list — pointing the model at the place where "ADR" was already correctly defined, instead of letting it rebuild the concept from raw ingredients.

## Bug: the same grain mistake, reappearing after the "fix"

Here's the harder finding. After fixing the ADR bug above, re-running the exact same question about cancellations — same fixed routing config, same correct business-logic notes explicitly warning "don't use `mart_aggregates` for this" — produced a **second wrong answer on a fresh run**, from the same grain confusion that showed up back in post 3 and post 4:

```sql
SELECT cr.month, ma.cancelled_count, cr.lost_revenue
FROM cancelled_rev cr
JOIN public_marts.mart_aggregates ma ON cr.month = ma.month
ORDER BY ma.cancelled_count DESC, cr.lost_revenue DESC
LIMIT 1;
```

`lost_revenue` was computed correctly this time (properly summed from `fct_reservation`, following the notes exactly). But the ranking still joined against `mart_aggregates.cancelled_count` — grained per accommodation, per month — without summing across accommodations first, so it ranked one property's single worst month, not the portfolio's. On a *different* run of this same question, with identical context and identical notes, the model got it right. Same prompt, same documentation, different generation, different answer.

> **The lesson for an Analytics Engineer, and the one this whole series has been circling:** an LLM is not a deterministic function of its prompt. The exact same context that produces a correct answer once can produce a subtly wrong one on the next run, for the exact bug you thought you'd already closed by writing it down in the notes. Documentation lowers the odds; it does not lock the door. The only thing that catches this reliably is checking the actual number against the actual data — every time, not just the first time.

## Round 2: pushing past the original 6 questions

The 6/6 result above uses chapter 07's original questions — the same 6 every post in this series has run, for a clean comparison. That's a narrow test. To actually probe how robust the fixed setup is, we ran 6 new questions, deliberately chosen to hit routing types the original 6 never touched, a real keyword-ambiguity case, and one question with nothing to do with this database at all. [Full transcript here.](transcript-expanded.md) Four more real bugs turned up immediately.

**Two confirmed, real coverage gaps.** "What's the average length of stay across all accommodations?" and "how far in advance do guests book?" both hit the fallback. Neither `avg_length_of_stay_nights` nor `avg_booking_window_days` — both real, validated `mart_kpi` columns — has ever had a routing entry pointing at it. An honest "I don't know" is the right behavior here, but it's still a gap worth closing.

**A false-positive SQL extraction, caused by the exact fix that closed a chapter 07 bug.** Asked which channel has the highest cancellation rate, the model gave a correct, honest answer: the table it could see has no channel column, so it can't compute that. Its response included the sentence *"...I can combine it **with** `mart_kpi` to answer your question."* The code that pulls SQL out of a model's response accepts anything starting with `SELECT` or `WITH` — a deliberate fix from chapter 07, so a real CTE-based query wouldn't get rejected. That same rule also matched the word "with" sitting in the middle of an ordinary English sentence, extracted `` with `mart_kpi` to answer your question. `` as if it were SQL, and handed it to Postgres — which correctly rejected it as a syntax error. The user-facing result: a confusing database error, for a question the model had actually answered correctly in plain English one line above it.

**A silent string-literal mismatch, with a coincidentally correct answer.** "Is occupancy higher during high season or low season?" produced a query comparing `season = 'High Season'` and `season = 'Low Season'` — but the real column only ever holds `'HIGH'` or `'LOW'`. Neither comparison can ever be true, so the query's `CASE` always falls through to its `ELSE` branch, regardless of the actual data. It happened to print the right answer here (`'LOW'` genuinely has more total nights: 23,171 vs. 11,068) — but that's luck, not logic; the exact same broken query would have confidently printed the wrong season if the real numbers had gone the other way. The same query also has a second, independent bug: it computes `total_reservations` by summing a `COUNT(*) OVER (PARTITION BY season)` window value once per row instead of taking it once per season, which squares the real count. HIGH season's real reservation count is 1,610 — the query returned 2,592,100, which is exactly 1,610².

**The most serious one: a rigid keyword caused the model to silently answer a different question than the one asked.** "Do stricter occupancy rules correlate with more cancellations?" was supposed to match `cancellation_by_dimension`, whose keyword is the literal phrase `"occupancy rules and cancellations"` — four words, including "and." The real question doesn't contain the word "and," so that keyword didn't match, and the question fell through to the generic `occupancy` type instead — a context containing only `mart_kpi`, with no occupancy-rules data in it anywhere. Rather than saying so, the model quietly substituted a *different, answerable* question — the Pearson correlation between `occupancy_rate_pct` and `cancellation_rate_pct` — computed it correctly, and presented the result as if it addressed the original question about occupancy *rules*. This is worse than every other failure in this series: it's not a wrong number, and it's not an honest refusal — it's a confident, syntactically correct answer to a question nobody asked, wearing the original question's name.

> **The lesson for an Analytics Engineer:** the honest refusals in this series (post 4's Q4, this round's channel question) are the *safe* failure mode — say what you don't have. The dangerous failure mode is a model that, given a plausible-but-wrong context, decides to answer a nearby question instead of the one it was actually asked, without flagging that it did so. A routing keyword that's too literal (requiring an exact word like "and") doesn't just cause a coverage gap — it can silently swap the question being answered for a different one that happens to be answerable with what's on hand.

One question in this round worked exactly as intended: "What is the capital of France?" hit the fallback immediately, no attempt to force a database answer out of an unrelated question. Coverage gaps produce honest refusals by design; it's the *near-miss* questions — ones almost, but not quite, matching a real routing pattern — that produce the interesting failures.

## What actually changed since post 1

| | Post 1 (no context) | Post 5, original 6 | Post 5, 6 new questions |
|---|---|---|---|
| Table/column hallucinations | 6 of 6 questions | 0 | 0 |
| Silent, fabricated numbers | 1 (a "70% occupancy," no query run) | 0 | 0 |
| Delivered transcript, verified correct | 0 of 6 | 6 of 6 | 2 of 6 |
| Honest fallback / refusal | 0 | 0 | 3 (2 real coverage gaps, 1 off-topic question) |
| Confidently wrong or silently-substituted answer | — | 2 found while building this, both fixed | 2 (a squared count + broken CASE; a silently answered different question) |

The middle and right columns are the same setup, the same fixed config, the same day. The only difference is which questions got asked. That's the actual finding of this round: robustness isn't a property of a system in the abstract, it's a property relative to the questions it's been tested against. Six new, reasonable questions — nothing adversarial, nothing designed to break it — found a bug in SQL extraction, a data-value assumption that happened to get lucky, and a routing keyword rigid enough to make the model quietly answer a different question than the one it was asked. None of that showed up in the original 6, not because it wasn't there, but because nothing had asked yet.

Good context and real access control turned "obviously broken, every time" into "usually right, occasionally and unpredictably wrong in a way that looks completely fine." That's real progress — and it's also exactly why "the agent answered without an error" can never be the bar. Budget for verification as a permanent line item, not a phase you finish, and budget for testing beyond whatever question set happened to be on hand when the system shipped.
