# Content experiment — CONTEXT_MODE=docs_full

Generated 2026-08-21T21:08:16.872096+00:00. Model: openai/gpt-oss-20b (via NVIDIA). Every question below gets the exact same context: the FULL public_marts schema, unfiltered by question -- no routing, no per-question scoping.

**Context given to the model for every question:**

```
Table: public_marts.mart_kpi
  Grain: one row per (accommodation_id, month), same as mart_aggregates. Derives the 6 business KPIs (occupancy, ADR, RevPAN, avg length of stay, avg booking window, cancellation rate) from mart_aggregates and stg_reservation.
  - accommodation_id (bigint) -- The accommodation this row rolls up.
  - month (date) -- First day of the calendar month this row covers (a date, not a formatted string).
  - occupancy_rate_pct (numeric) -- 100 * occupied_nights / available_nights, from mart_aggregates. NULL (not 0) when available_nights is 0.
  - adr (numeric) -- Average Daily Rate: revenue / occupied_nights. Both inputs are prorated across month boundaries -- see mart_aggregates' revenue/occupied_nights docs. NULL (not 0) when occupied_nights is 0, meaning there's nothing to average, not a $0 rate.
  - revpan (numeric) -- Revenue Per Available Night: revenue / available_nights. See mart_aggregates' revenue/available_nights docs for the proration logic. NULL (not 0) when available_nights is 0.
  - avg_length_of_stay_nights (numeric) -- Average nights per confirmed reservation. Sourced directly from stg_reservation (not mart_aggregates) and NOT prorated across month boundaries -- attributed to the month of the reservation's start_date, same as reservation_count/cancelled_count.
  - avg_booking_window_days (numeric) -- Average number of days between when a guest booked a reservation and their check-in date (`start_date - booked_at`), for confirmed reservations starting in this month. **Do not compute this from `reservation.created_at`.** `created_at` is a technical row-insert timestamp, not the date the guest actually made the booking — for this project's seed data, most reservations were bulk-inserted long after their real booking date, and `start_date - created_at` comes out negative for the majority of historical rows. `booked_at` is a dedicated column, backfilled to a plausible 1-60 days before `start_date`, generated server-side for new reservations, and protected by a `CHECK (booked_at <= start_date)` constraint so this can't regress.
  - cancellation_rate_pct (numeric) -- 100 * cancelled_count / reservation_count, from mart_aggregates. Both counts are attributed to the month of start_date. NULL (not 0) when reservation_count is 0.

Table: public_marts.mart_aggregates
  Grain: one row per (accommodation_id, month). available_nights and occupied_nights are prorated for stays/listings that cross a month boundary. revenue/reservation_count/cancelled_count are attributed to the month a reservation's start_date falls in.
  - accommodation_id (bigint) -- The accommodation this row rolls up.
  - month (date) -- First day of the calendar month this row covers.
  - available_nights (integer) -- Total calendar capacity for this accommodation in this month, bounded below by listed_at.
  - revenue (numeric) -- Revenue prorated by night across month boundaries, same technique as occupied_nights (per-night rate x overlapping nights in this month). Must stay consistent with occupied_nights' proration -- attributing full revenue to a reservation's start month broke ADR/RevPAN for cross-month stays (found and fixed in chapter 09).
  - occupied_nights (bigint) -- Confirmed nights booked in this month. Always <= available_nights -- if this test fails, it means the month_spine/proration logic has regressed (this exact inconsistency was found and fixed once already, see chapter 09).
  - reservation_count (bigint) -- Count of all reservations (confirmed and cancelled) starting in this month. Not prorated -- a booking is one event, attributed to its start_date's month.
  - cancelled_count (bigint) -- Subset of reservation_count with status = 'cancelled', same month attribution (start_date, not cancelled_at).

Table: public_marts.fct_reservation
  Grain: one row per reservation. A governed, PII-scrubbed fact mart -- built specifically so the agent (chapters 12-13) can answer row-level and channel/season-breakdown questions without ever touching stg_reservation (which carries guest_name and has no access-control boundary). Deliberately excludes property_manager_id -- join to stg_accommodation for the CURRENT manager if needed, rather than trusting this table's reservation-level snapshot.
  - reservation_id (bigint) -- Primary key.
  - accommodation_id (bigint) -- The accommodation booked.
  - start_date (date) -- First night of the stay.
  - end_date (date) -- Checkout date (exclusive).
  - nights (smallint) -- end_date - start_date.
  - status (text) -- confirmed or cancelled.
  - channel (text) -- AIRBNB, BOOKING, or EXPEDIA.
  - season (text) -- HIGH or LOW. A frozen snapshot computed server-side at booking time from season_calendar -- if season_calendar changes later, existing reservations keep their original value.
  - total_price (numeric) -- GENERATED ALWAYS, populated regardless of status -- a cancelled reservation still has a non-null total_price (needed for lost-revenue questions). Any real-revenue question (e.g. by channel or season) must filter status = 'confirmed' explicitly.
  - cancelled_at (timestamp with time zone) -- Null unless status = 'cancelled'.
  - booked_at (date) -- The business date this reservation was actually booked (not a technical timestamp) -- always <= start_date. See mart_kpi's avg_booking_window_days docs for why created_at can't be used for this instead.

Table: public_marts.dim_accommodation
  Grain: one row per accommodation. A governed dimension mart -- flattens occupancy_rules (verified 1:1) and pre-aggregates accommodation_cost (1:many, summed here so no consumer has to join the raw multi-row cost table) so the agent (chapters 12-13) never touches stg_accommodation/stg_occupancy_rules/stg_accommodation_cost directly. Deliberately excludes room_count and postal_code/ neighborhood -- both have documented data quality issues (see _sources.yml) that would mislead an agent using them at face value.
  - accommodation_id (bigint) -- Primary key.
  - property_manager_id (bigint) -- The CURRENT manager of this accommodation (not a booking-time snapshot -- see fct_reservation for that distinction).
  - accommodation_type (text) -- FLAT, VILLA, or ROOM.
  - city (text) -- City, from a fixed canonical list.
  - country (text) -- Country, same canonical list as property_manager.country.
  - listed_at (date) -- The business date this accommodation became available for booking -- see mart_aggregates' available_nights docs for why created_at could not be used for this instead.
  - min_stay_nights (smallint) -- Minimum nights per stay, enforced by the reservation engine at booking time.
  - max_stay_nights (smallint) -- Maximum nights per stay.
  - max_guests (smallint) -- Maximum guest count, enforced by the reservation engine at booking time.
  - no_smoking (boolean) -- Whether smoking is disallowed.
  - total_cost (numeric) -- SUM(amount) from stg_accommodation_cost, lifetime-to-date. 0 (not null) for an accommodation with no cost rows. Pre-aggregated here specifically so a consumer never joins the raw cost table directly and risks a fan-out -- see agent_routing.yml's adr_vs_cost notes for the bug this prevents.

```

## Question 1

**Question:** What is the total RevPAN for property manager 20, broken down by each of their accommodations?

**Raw model response:**

````
```sql
/* Total RevPAN for each accommodation belonging to property manager 20 */
SELECT
    d.accommodation_id,
    SUM(mk.revpan) AS total_revpan
FROM public_marts.mart_kpi            AS mk
JOIN public_marts.dim_accommodation  AS d
  ON mk.accommodation_id = d.accommodation_id
WHERE d.property_manager_id = 20
GROUP BY d.accommodation_id
ORDER BY total_revpan DESC;
```
````

**Extracted SQL:**

```sql
/* Total RevPAN for each accommodation belonging to property manager 20 */
SELECT
    d.accommodation_id,
    SUM(mk.revpan) AS total_revpan
FROM public_marts.mart_kpi            AS mk
JOIN public_marts.dim_accommodation  AS d
  ON mk.accommodation_id = d.accommodation_id
WHERE d.property_manager_id = 20
GROUP BY d.accommodation_id
ORDER BY total_revpan DESC;
```

**Execution outcome:**

| accommodation_id | total_revpan |
|---|---|
| 220 | 1006.01 |
| 227 | 730.48 |
| 230 | 687.94 |
| 233 | 642.96 |
| 222 | 560.65 |
| 235 | 495.37 |
| 228 | 491.04 |
| 236 | 449.19 |
| 229 | 423.37 |
| 231 | 422.63 |
| 221 | 402.77 |
| 237 | 304.68 |
| 224 | 303.53 |
| 218 | 297.50 |
| 225 | 244.61 |
| 232 | 224.67 |
| 223 | 215.00 |
| 219 | 186.07 |
| 226 | 134.66 |
| 234 | 91.38 |
---

## Question 2

**Question:** How many cancelled reservations has accommodation 15 had?

**Raw model response:**

````
```sql
SELECT COUNT(*) AS cancelled_reservations
FROM public_marts.fct_reservation
WHERE accommodation_id = 15
  AND status = 'cancelled';
```
````

**Extracted SQL:**

```sql
SELECT COUNT(*) AS cancelled_reservations
FROM public_marts.fct_reservation
WHERE accommodation_id = 15
  AND status = 'cancelled';
```

**Execution outcome:**

| cancelled_reservations |
|---|
| 1 |
---

## Question 3

**Question:** What is the occupancy rate for property manager 9 in August 2026?

**Raw model response:**

````
```sql
/* Occupancy rate for the properties managed by property_manager_id = 9
   in August 2026 (first day of the month).  The rate is calculated as
   100 × (total occupied nights) ÷ (total available nights) across all
   that manager's accommodations. */
SELECT
    100.0 * SUM(ma.occupied_nights) / NULLIF(SUM(ma.available_nights), 0) AS occupancy_rate_pct
FROM public_marts.mart_aggregates AS ma
JOIN public_marts.dim_accommodation AS da
  ON ma.accommodation_id = da.accommodation_id
WHERE da.property_manager_id = 9
  AND ma.month = '2026-08-01';
```

This returns a single numeric value (or `NULL` if the manager had no available nights in that month).
````

**Extracted SQL:**

```sql
/* Occupancy rate for the properties managed by property_manager_id = 9
   in August 2026 (first day of the month).  The rate is calculated as
   100 × (total occupied nights) ÷ (total available nights) across all
   that manager's accommodations. */
SELECT
    100.0 * SUM(ma.occupied_nights) / NULLIF(SUM(ma.available_nights), 0) AS occupancy_rate_pct
FROM public_marts.mart_aggregates AS ma
JOIN public_marts.dim_accommodation AS da
  ON ma.accommodation_id = da.accommodation_id
WHERE da.property_manager_id = 9
  AND ma.month = '2026-08-01';
```

**Execution outcome:**

| occupancy_rate_pct |
|---|
| 11.1290322580645161 |
---

## Question 4

**Question:** Which accommodations have an ADR lower than their normalized daily cost over the last 3 months?

**Raw model response:**

````
```sql
/* Accommodations whose ADR is lower than their normalized daily cost
   for the most recent three calendar months.  We compute a "normalized
   cost per day" as total lifetime cost divided by the sum of
   available nights in the same period.  The ADR is taken as the overall
   revenue divided by the overall occupied nights in that period. */

WITH period AS (
  -- First day of the month two months ago (inclusive)
  SELECT date_trunc('month', current_date) - interval '2 months' AS start_month
),

monthly AS (
  SELECT a.accommodation_id,
         sum(a.revenue)          AS tot_revenue,
         sum(a.occupied_nights)   AS tot_occ_nights,
         sum(a.available_nights)  AS tot_avail_nights
  FROM public_marts.mart_aggregates a
  JOIN period p
    ON a.month >= p.start_month                -- last three months
   AND a.month <  date_trunc('month', current_date) + interval '1 month'
  GROUP BY a.accommodation_id
),

costs AS (
  SELECT d.accommodation_id,
         d.total_cost
  FROM public_marts.dim_accommodation d
)

SELECT m.accommodation_id,
       m.tot_revenue / NULLIF(m.tot_occ_nights,0)      AS overall_adr,
       c.total_cost / NULLIF(m.tot_avail_nights,0)    AS normalised_daily_cost
FROM monthly m
JOIN costs c
  ON c.accommodation_id = m.accommodation_id
WHERE m.tot_occ_nights > 0            -- avoid division‑by‑zero
  AND m.tot_avail_nights > 0
  AND c.total_cost IS NOT NULL
  AND m.tot_revenue / NULLIF(m.tot_occ_nights,0)
        < c.total_cost / NULLIF(m.tot_avail_nights,0);
```

This returns the set of `accommodation_id`s (with their ADR and the corresponding normalized daily cost) for which the ADR is strictly lower than the cost per available night over the last three months.
````

**Extracted SQL:**

```sql
/* Accommodations whose ADR is lower than their normalized daily cost
   for the most recent three calendar months.  We compute a "normalized
   cost per day" as total lifetime cost divided by the sum of
   available nights in the same period.  The ADR is taken as the overall
   revenue divided by the overall occupied nights in that period. */

WITH period AS (
  -- First day of the month two months ago (inclusive)
  SELECT date_trunc('month', current_date) - interval '2 months' AS start_month
),

monthly AS (
  SELECT a.accommodation_id,
         sum(a.revenue)          AS tot_revenue,
         sum(a.occupied_nights)   AS tot_occ_nights,
         sum(a.available_nights)  AS tot_avail_nights
  FROM public_marts.mart_aggregates a
  JOIN period p
    ON a.month >= p.start_month                -- last three months
   AND a.month <  date_trunc('month', current_date) + interval '1 month'
  GROUP BY a.accommodation_id
),

costs AS (
  SELECT d.accommodation_id,
         d.total_cost
  FROM public_marts.dim_accommodation d
)

SELECT m.accommodation_id,
       m.tot_revenue / NULLIF(m.tot_occ_nights,0)      AS overall_adr,
       c.total_cost / NULLIF(m.tot_avail_nights,0)    AS normalised_daily_cost
FROM monthly m
JOIN costs c
  ON c.accommodation_id = m.accommodation_id
WHERE m.tot_occ_nights > 0            -- avoid division‑by‑zero
  AND m.tot_avail_nights > 0
  AND c.total_cost IS NOT NULL
  AND m.tot_revenue / NULLIF(m.tot_occ_nights,0)
        < c.total_cost / NULLIF(m.tot_avail_nights,0);
```

**Execution outcome:**

| accommodation_id | overall_adr | normalised_daily_cost |
|---|---|---|
---

## Question 5

**Question:** Which month had the most cancelled reservations, and what is the lost revenue for that month?

**Raw model response:**

````
```sql
-- 1. Determine the month with most cancelled reservations
WITH max_cancelled_month AS (
    SELECT month,
           cancelled_count
    FROM public_marts.mart_aggregates
    ORDER BY cancelled_count DESC
    LIMIT 1
),

-- 2. Compute the lost revenue from those cancelled reservations
lost_rev AS (
    SELECT SUM(fr.total_price) AS lost_revenue
    FROM public_marts.fct_reservation fr
    JOIN max_cancelled_month mc
      ON date_trunc('month', fr.start_date)::date = mc.month
    WHERE fr.status = 'cancelled'
)

SELECT
    mc.month   AS month_with_most_cancellations,
    mc.cancelled_count,
    lr.lost_revenue
FROM max_cancelled_month mc
CROSS JOIN lost_rev lr;
```

**Explanation**

* **`max_cancelled_month`** pulls the month (first day of the month) with the highest `cancelled_count` from `mart_aggregates`.  
* **`lost_rev`** sums the `total_price` for every reservation where `status = 'cancelled'` whose `start_date` falls in that same month.  
* The final SELECT returns the month, the number of cancelled reservations, and the total lost revenue for that month.

The query is read‑only, self‑contained, and uses only the tables described.
````

**Extracted SQL:**

```sql
-- 1. Determine the month with most cancelled reservations
WITH max_cancelled_month AS (
    SELECT month,
           cancelled_count
    FROM public_marts.mart_aggregates
    ORDER BY cancelled_count DESC
    LIMIT 1
),

-- 2. Compute the lost revenue from those cancelled reservations
lost_rev AS (
    SELECT SUM(fr.total_price) AS lost_revenue
    FROM public_marts.fct_reservation fr
    JOIN max_cancelled_month mc
      ON date_trunc('month', fr.start_date)::date = mc.month
    WHERE fr.status = 'cancelled'
)

SELECT
    mc.month   AS month_with_most_cancellations,
    mc.cancelled_count,
    lr.lost_revenue
FROM max_cancelled_month mc
CROSS JOIN lost_rev lr;
```

**Execution outcome:**

| month_with_most_cancellations | cancelled_count | lost_revenue |
|---|---|---|
| 2026-02-01 | 2 | 62284.22 |
---

## Question 6

**Question:** How many accommodations does property manager 26 have?

**Raw model response:**

````
```sql
SELECT COUNT(*) AS accommodation_count
FROM public_marts.dim_accommodation
WHERE property_manager_id = 26;
```
This query returns the number of accommodations managed by property manager 26.
````

**Extracted SQL:**

```sql
SELECT COUNT(*) AS accommodation_count
FROM public_marts.dim_accommodation
WHERE property_manager_id = 26;
```

**Execution outcome:**

| accommodation_count |
|---|
| 20 |
---

## Resultados en tabla

| Pregunta | Columnas devueltas | Resultado visible |
|---|---|---|
| 1. RevPAN del manager 20 | `accommodation_id`, `total_revpan` | 20 filas; máximo: 220 → 1006.01 |
| 2. Cancelaciones del alojamiento 15 | `cancelled_reservations` | **1** |
| 3. Ocupación del manager 9 en agosto 2026 | `occupancy_rate_pct` | **11.1290322580645161%** |
| 4. ADR menor que coste normalizado | `accommodation_id`, `overall_adr`, `normalised_daily_cost` | **0 filas** |
| 5. Mes con más cancelaciones | `month_with_most_cancellations`, `cancelled_count`, `lost_revenue` | **2026-02-01 · 2 · 62,284.22** |
| 6. Alojamientos del manager 26 | `accommodation_count` | **20** |
