# 9 Real Errors and How We Fixed Them

Every bug below actually happened while building this project — a Postgres-backed property management system with a dbt semantic layer and a text-to-SQL agent on top. Nothing here is staged for the post; each one is pulled from the project's own [error log](https://github.com/moradabaz/pms-semantic-layer/blob/main/agent_stack/chapters/08-dbt-project-setup/errors-log.md) and chapter notes. Each entry follows the same format: what broke, where else this exact shape of bug shows up, and what we changed.

They're grouped by the layer they lived in, because the layer tells you where to look for the same bug in your own project.

---

## The staging layer

dbt's staging layer is supposed to be a boring, 1:1 mirror of a raw source: rename a column, cast a type, nothing more. Both bugs here came from breaking that rule "just a little."

### 1. A `JOIN` inside a staging model

**What broke:** `stg_accommodation` (a staging model over the `accommodation` table) had a `LEFT JOIN` to `property_manager`, added to pull in `company_name` for convenience.

```sql
select
    a.accommodation_id,
    pm.company_name
from {{ source('pms', 'accommodation') }} a
left join {{ source('pms', 'property_manager') }} pm
    on pm.property_manager_id = a.property_manager_id
```

**Why it's a real problem, not a style nit:** a staging model that joins another source stops being a faithful mirror of the table it's named after — a bug in `property_manager`, or a future change to it, can now silently change what `stg_accommodation` returns. It also hides a fan-out risk one layer earlier than anyone would think to check: if `property_manager` ever has more than one row per `property_manager_id`, this join silently duplicates accommodation rows, and nothing at the staging layer would tell you.

**Where this shows up in other projects:** any staging/bronze layer where "just one more join for convenience" creeps in. The tell is a staging model that references more than one `source()`.

**The fix:** `stg_accommodation` stays scoped to the `accommodation` source, full stop. A mart that needs `company_name` alongside accommodation data joins two staging models explicitly, at the mart layer, where joins are supposed to happen.

### 2. A business rule filtered out data one layer too early

**What broke:** `stg_reservation` had `WHERE status = 'confirmed'`, added on the (reasonable-sounding) theory that cancelled reservations should never count toward revenue, so why not filter them out immediately.

**Why it's a real problem:** "filter it out as early as possible" is the wrong instinct here. The rule is specific to *revenue* — but a `WHERE` in staging doesn't just apply a rule, it deletes information for every downstream consumer, including ones the rule was never meant for. A cancellation-rate KPI built later needed exactly the cancelled rows this filter had thrown away.

**Where this shows up:** anywhere a business rule that applies to *one* metric gets hoisted into a shared layer everyone reads from. If applying a rule early would make some other valid downstream use case impossible, it's in the wrong place.

**The fix:** staging keeps every row, unfiltered, always. `status = 'confirmed'` gets applied explicitly inside whichever mart actually computes revenue.

---

## The mart layer: proration and time

The next four bugs all involve months. Any metric with a monthly grain built from reservations that can span a month boundary has to prorate consistently — get one side of that consistency wrong and the numbers still look plausible.

### 3. A technical timestamp used as a business anchor

**What broke:** `mart_aggregates` needed to know how many nights an accommodation had been available for, anchored to when it was listed. The obvious column, `accommodation.created_at`, turned out to be a lie for this purpose: every accommodation had been bulk-inserted in a single seed run, so `created_at` was really "when the seed script ran," not "when this listing went live." 82% of reservations started *before* their own accommodation's `created_at`.

**Where this shows up:** any table with a `created_at` that's actually a row-insert timestamp, used as if it were a meaningful business date. This is the single most common trap in this project — it happens again in bug #4.

**The fix:** added `accommodation.listed_at`, backfilled to 30 days before each accommodation's earliest real reservation. A new column with real meaning, not a repurposed technical field.

### 4. The same pattern, again, for booking windows

**What broke:** the "booking window" KPI (days between booking and check-in) used `start_date - reservation.created_at`. For historical reservations, this came out **negative** — a reservation "booked" after it had already started.

**Why it happened twice:** same root cause as #3, different column. `created_at` on any bulk-seeded table is a row-insert artifact, not a fact about the business.

**The fix:** added `reservation.booked_at`, backfilled to 1–60 days before `start_date`, with a database-level `CHECK (booked_at <= start_date)` constraint — so this specific failure mode can't come back, no matter what inserts the row later.

> **Common mistake:** if a "since X" or "days between X and Y" calculation ever produces a negative number, the first suspect is a `created_at`/`updated_at` column standing in for a business date it was never designed to represent.

### 5. Occupied nights only counted a reservation's start month

**What broke:** a reservation spanning, say, August 28 – September 9 had all of its nights attributed to August in `mart_aggregates`, none to September — because the first version of the model grouped by the reservation's `start_date` month instead of splitting nights across every month the stay actually touches.

**The fix:** cross-join every reservation against a month spine, and compute `LEAST(end_date, month_end) - GREATEST(start_date, month_start)` per month — the overlap between the stay and that specific month, clamped to zero.

**Where this shows up:** any monthly rollup built from date-range facts (bookings, subscriptions, leases) that don't happen to align with calendar month boundaries.

### 6. Revenue prorated differently than the nights it was divided by

**What broke:** occupied nights were correctly split across months (once #5 was fixed), but revenue was still attributed entirely to the reservation's start month. For a cross-month stay, this meant ADR and RevPAN — both of which divide revenue by nights — used a numerator and denominator counted over *different* sets of nights, silently producing wrong ratios.

**The fix:** revenue is now prorated with the exact same technique as nights — per-night rate × nights overlapping that month.

> **Key lesson:** if two columns feed the same ratio (revenue ÷ nights, cost ÷ time, spend ÷ clicks), they need the *same* proration logic. Fixing one side and not the other doesn't just leave a bug — it manufactures a new, more subtle one.

### 7. Available nights capped at "today," inconsistent with confirmed future bookings

**What broke:** `available_nights` for the current month was capped at `current_date`, on the assumption that nights in the future "aren't available yet." But `occupied_nights` already counted confirmed nights in the future (most bookings are made well ahead of the stay) — so for the current month, occupied nights could exceed available nights, an invariant that should never break.

**The fix:** removed the `current_date` cap entirely. A month's capacity exists whether or not those calendar days have elapsed — occupancy and availability need to be measured on the same timeline.

**Where this shows up:** any capacity metric that mixes "elapsed time" logic with "confirmed future commitments" logic without checking they agree on what "available" means.

---

## Data generation

### 8. A backfill fixed the past but not the future

**What broke:** after fixing #3 (`listed_at`), the synthetic data simulator kept running for days, generating new reservations with random start dates — with no awareness that `listed_at` now existed. It eventually generated a reservation starting *before* its own accommodation's `listed_at`, violating an invariant that had only ever held by historical coincidence, not by design.

**Why it's worth calling out separately from #3:** a one-time backfill that establishes an invariant is not the same as an invariant that's *enforced* going forward. Fixing the historical data and forgetting the thing that generates new data is a gap that can sit invisible for a long time — this one surfaced days later, and only because an unrelated freshness check happened to restart the generator and produce fresh rows.

**The fix:** the generator's date-range function now takes `listed_at` as a parameter and clamps its earliest possible date to it. Fixed at the source that creates the data, not with a wider tolerance on the mart-layer test that caught it.

---

## The agent layer

### 9. A confidently wrong formula, built from an incomplete (but real) context

**What broke:** a text-to-SQL agent, given real but incomplete schema context, computed "ADR" as `revenue / available_nights` — that's RevPAN's formula, not ADR's. Syntactically perfect, no error, wrong for every accommodation. This one gets the full treatment — the exact SQL, why good context wasn't enough, and a second bug that came back after the first fix — in [post 5 of this series](../05-full-context-and-scope/post.md), so it's not repeated here.

---

## The pattern across all nine

Five of these bugs (#3, #4, #5, #6, #7) are really one lesson wearing different clothes: **a technical column (a timestamp, a cap, an unprorated total) was trusted to mean something it didn't.** Two more (#1, #2) are the staging-layer version of the same idea — a column or a filter placed one layer earlier than its actual meaning belonged. #8 is what happens when a fix addresses the data but not the thing that keeps generating more of it. And #9 is the same root cause showing up one layer higher, in the prompt handed to an LLM instead of the SQL handed to a database.

None of these were hallucinations in the sense of an invented table or a fabricated number — they were all syntactically valid, ran without error, and looked reasonable. That's what makes them worth writing down: the failure mode that doesn't throw an error is the one that needs a habit of checking, not a one-time fix.
