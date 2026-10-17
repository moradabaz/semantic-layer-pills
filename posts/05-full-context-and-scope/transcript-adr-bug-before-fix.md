# Chapter 13 — Transcript: the live agent

Generated 2026-08-21T21:26:20.127814+00:00. Model: openai/gpt-oss-20b (via NVIDIA). Unlike chapter 07, each question is routed to a matched question_type from agent_routing.yml and given a scoped schema context before the model sees it. Same 6 questions as chapter 07's transcript, for a direct before/after comparison.

## Question 1

**Question:** What is the total RevPAN for property manager 20, broken down by each of their accommodations?

**Matched question_type:** revenue_per_property_manager

**Context given to the model:**

```
Question type: revenue_per_property_manager

Table: public_marts.mart_kpi
  - accommodation_id (bigint)
  - month (date)
  - occupancy_rate_pct (numeric)
  - adr (numeric)
  - revpan (numeric)
  - avg_length_of_stay_nights (numeric)
  - avg_booking_window_days (numeric)
  - cancellation_rate_pct (numeric)

Table: public_marts.dim_accommodation
  - accommodation_id (bigint)
  - property_manager_id (bigint)
  - accommodation_type (text)
  - city (text)
  - country (text)
  - listed_at (date)
  - min_stay_nights (smallint)
  - max_stay_nights (smallint)
  - max_guests (smallint)
  - no_smoking (boolean)
  - total_cost (numeric)

Join keys:
  - mart_kpi.accommodation_id = dim_accommodation.accommodation_id

Notes: mart_kpi/mart_aggregates only have accommodation_id, not property_manager_id -- join to dim_accommodation to filter/group by property manager. dim_accommodation.property_manager_id is the CURRENT manager. A historical variant of this question ("who managed this at the time of booking X") is deliberately out of scope: fct_reservation excludes property_manager_id on purpose (see its own docs), and no mart currently exposes the booking-time snapshot -- that question should hit the fallback, not reach into stg_reservation directly.
```

**Raw model response:**

````
```sql
-- Total RevPAN per accommodation for property manager 20
SELECT
    a.accommodation_id,
    SUM(k.revpan) AS total_revpan
FROM
    public_marts.mart_kpi          AS k
JOIN
    public_marts.dim_accommodation AS a
      ON k.accommodation_id = a.accommodation_id
WHERE
    a.property_manager_id = 20
GROUP BY
    a.accommodation_id
ORDER BY
    total_revpan DESC;   -- optional: highest revenue first
```

This query sums the `revpan` values from `mart_kpi` for each accommodation belonging to property manager 20, grouping by `accommodation_id`. Replace the `ORDER BY` clause with any desired ordering.
````

**Extracted SQL:**

```sql
-- Total RevPAN per accommodation for property manager 20
SELECT
    a.accommodation_id,
    SUM(k.revpan) AS total_revpan
FROM
    public_marts.mart_kpi          AS k
JOIN
    public_marts.dim_accommodation AS a
      ON k.accommodation_id = a.accommodation_id
WHERE
    a.property_manager_id = 20
GROUP BY
    a.accommodation_id
ORDER BY
    total_revpan DESC;   -- optional: highest revenue first
```

**Execution outcome:**

| Estado | Columnas | Filas devueltas |
|---|---|---|
| Success | `['accommodation_id', 'total_revpan']` | [(220, Decimal('1006.01')), (227, Decimal('730.48')), (230, Decimal('687.94')), (233, Decimal('642.96')), (222, Decimal('560.65')), (235, Decimal('495.37')), (228, Decimal('491.04')), (236, Decimal('449.19')), (229, Decimal('423.37')), (231, Decimal('422.63')), (221, Decimal('402.77')), (237, Decimal('304.68')), (224, Decimal('303.53')), (218, Decimal('297.50')), (225, Decimal('244.61')), (232, Decimal('224.67')), (223, Decimal('215.00')), (219, Decimal('186.07')), (226, Decimal('134.66')), (234, Decimal('91.38'))]. |


---

## Question 2

**Question:** How many cancelled reservations has accommodation 15 had?

**Matched question_type:** cancelled_reservations_list

**Context given to the model:**

```
Question type: cancelled_reservations_list

Table: public_marts.fct_reservation
  - reservation_id (bigint)
  - accommodation_id (bigint)
  - start_date (date)
  - end_date (date)
  - nights (smallint)
  - status (text)
  - channel (text)
  - season (text)
  - total_price (numeric)
  - cancelled_at (timestamp with time zone)
  - booked_at (date)

Notes: Row-level listing OR a per-accommodation/lifetime count (e.g. "how many cancelled reservations has accommodation X had") -- either way, query fct_reservation directly with WHERE status = 'cancelled' (COUNT(*) for a count, no aggregation for a listing). Do not use mart_aggregates/mart_kpi for this: they only carry cancelled_count pre-aggregated per accommodation+month, not a lifetime total or the individual reservations.
```

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

| Estado | Columnas | Filas devueltas |
|---|---|---|
| Success | `['cancelled_reservations']` | [(1,)]. |


---

## Question 3

**Question:** What is the occupancy rate for property manager 9 in August 2026?

**Matched question_type:** revenue_per_property_manager

**Context given to the model:**

```
Question type: revenue_per_property_manager

Table: public_marts.mart_kpi
  - accommodation_id (bigint)
  - month (date)
  - occupancy_rate_pct (numeric)
  - adr (numeric)
  - revpan (numeric)
  - avg_length_of_stay_nights (numeric)
  - avg_booking_window_days (numeric)
  - cancellation_rate_pct (numeric)

Table: public_marts.dim_accommodation
  - accommodation_id (bigint)
  - property_manager_id (bigint)
  - accommodation_type (text)
  - city (text)
  - country (text)
  - listed_at (date)
  - min_stay_nights (smallint)
  - max_stay_nights (smallint)
  - max_guests (smallint)
  - no_smoking (boolean)
  - total_cost (numeric)

Join keys:
  - mart_kpi.accommodation_id = dim_accommodation.accommodation_id

Notes: mart_kpi/mart_aggregates only have accommodation_id, not property_manager_id -- join to dim_accommodation to filter/group by property manager. dim_accommodation.property_manager_id is the CURRENT manager. A historical variant of this question ("who managed this at the time of booking X") is deliberately out of scope: fct_reservation excludes property_manager_id on purpose (see its own docs), and no mart currently exposes the booking-time snapshot -- that question should hit the fallback, not reach into stg_reservation directly.
```

**Raw model response:**

````
```sql
-- Occupancy rate for property manager 9 in August 2026
SELECT
    ROUND(AVG(kpi.occupancy_rate_pct), 2) AS occupancy_rate_pct
FROM public_marts.mart_kpi AS kpi
JOIN public_marts.dim_accommodation AS dim
      ON kpi.accommodation_id = dim.accommodation_id
WHERE dim.property_manager_id = 9
  AND kpi.month >= DATE '2026-08-01'
  AND kpi.month <  DATE '2026-09-01';
```
````

**Extracted SQL:**

```sql
-- Occupancy rate for property manager 9 in August 2026
SELECT
    ROUND(AVG(kpi.occupancy_rate_pct), 2) AS occupancy_rate_pct
FROM public_marts.mart_kpi AS kpi
JOIN public_marts.dim_accommodation AS dim
      ON kpi.accommodation_id = dim.accommodation_id
WHERE dim.property_manager_id = 9
  AND kpi.month >= DATE '2026-08-01'
  AND kpi.month <  DATE '2026-09-01';
```

**Execution outcome:**

| Estado | Columnas | Filas devueltas |
|---|---|---|
| Success | `['occupancy_rate_pct']` | [(Decimal('11.13'),)]. |


---

## Question 4

**Question:** Which accommodations have an ADR lower than their normalized daily cost over the last 3 months?

**Matched question_type:** adr_vs_cost

**Context given to the model:**

```
Question type: adr_vs_cost

Table: public_marts.mart_aggregates
  - accommodation_id (bigint)
  - month (date)
  - available_nights (integer)
  - revenue (numeric)
  - occupied_nights (bigint)
  - reservation_count (bigint)
  - cancelled_count (bigint)

Table: public_marts.dim_accommodation
  - accommodation_id (bigint)
  - property_manager_id (bigint)
  - accommodation_type (text)
  - city (text)
  - country (text)
  - listed_at (date)
  - min_stay_nights (smallint)
  - max_stay_nights (smallint)
  - max_guests (smallint)
  - no_smoking (boolean)
  - total_cost (numeric)

Join keys:
  - mart_aggregates.accommodation_id = dim_accommodation.accommodation_id

Notes: dim_accommodation.total_cost is already pre-aggregated (lifetime-to-date, one row per accommodation) -- joining it to mart_aggregates carries no fan-out risk. But a GRAIN MISMATCH remains even so: total_cost has no month dimension, while mart_aggregates/mart_kpi are monthly. Comparing a recent-months ADR against a lifetime cost figure needs the cost side normalized the same way:
  1. SUM(mart_aggregates.available_nights) GROUP BY accommodation_id
     -- total available nights since listed_at (lifetime, for the
     cost side only -- not the same window as the ADR average above).
  2. dim_accommodation.total_cost -- already a single lifetime
     total per accommodation_id, no aggregation needed on this side.
  3. normalized_cost_per_night = (2) / (1) -- cost per available
     night, directly comparable to a lifetime RevPAN figure if one
     is needed, but NOT directly comparable to a recent-3-month ADR
     average -- the two still cover different time windows. Treat
     any ADR-vs-cost comparison as approximate for that reason, and
     say so in the answer rather than presenting it as exact.
A plain total_cost vs SUM(revenue) comparison (no per-night normalization) is NOT an acceptable substitute -- it still favors older/larger accommodations regardless of profitability.
```

**Raw model response:**

````
**SQL**

```sql
WITH
    /* Total available nights for each accommodation (every month recorded so far) */
    lifetime_available AS (
        SELECT accommodation_id,
               SUM(available_nights) AS total_available_nights
        FROM public_marts.mart_aggregates
        GROUP BY accommodation_id
    ),

    /* ADR for the most recent three calendar months (inclusive of the last full month) */
    adr_last_3_months AS (
        SELECT accommodation_id,
               SUM(revenue) / SUM(available_nights) AS adr
        FROM public_marts.mart_aggregates
        WHERE month >= date_trunc('month', current_date - INTERVAL '3 months')
          AND month <  date_trunc('month', current_date)
        GROUP BY accommodation_id
    ),

    /* Normalised cost per available night (lifetime cost ÷ lifetime available nights) */
    normalised_cost AS (
        SELECT d.accommodation_id,
               d.total_cost / l.total_available_nights AS cost_per_night
        FROM public_marts.dim_accommodation d
        JOIN lifetime_available l ON l.accommodation_id = d.accommodation_id
    )

SELECT  a.accommodation_id,
        a.adr,
        n.cost_per_night
FROM   adr_last_3_months      a
JOIN   normalised_cost        n USING (accommodation_id)
WHERE  a.adr < n.cost_per_night
ORDER BY a.accommodation_id;
```

**What this does**

1. **`lifetime_available`** aggregates all `available_nights` ever recorded for each accommodation.  
2. **`adr_last_3_months`** computes the average daily rate for the last 3 *calendar* months.  
3. **`normalised_cost`** divides each accommodation’s lifetime `total_cost` by its lifetime available nights, giving a cost per available night.  
4. The final `SELECT` lists every accommodation whose ADR (last‑3‑month average) is *lower* than its normalised cost per night.  

> **Caveat** – Because the ADR window is the last 3 months while the cost window covers the entire lifetime since the property was listed, the comparison is only approximate and should be interpreted with that context in mind.
````

**Extracted SQL:**

```sql
WITH
    /* Total available nights for each accommodation (every month recorded so far) */
    lifetime_available AS (
        SELECT accommodation_id,
               SUM(available_nights) AS total_available_nights
        FROM public_marts.mart_aggregates
        GROUP BY accommodation_id
    ),

    /* ADR for the most recent three calendar months (inclusive of the last full month) */
    adr_last_3_months AS (
        SELECT accommodation_id,
               SUM(revenue) / SUM(available_nights) AS adr
        FROM public_marts.mart_aggregates
        WHERE month >= date_trunc('month', current_date - INTERVAL '3 months')
          AND month <  date_trunc('month', current_date)
        GROUP BY accommodation_id
    ),

    /* Normalised cost per available night (lifetime cost ÷ lifetime available nights) */
    normalised_cost AS (
        SELECT d.accommodation_id,
               d.total_cost / l.total_available_nights AS cost_per_night
        FROM public_marts.dim_accommodation d
        JOIN lifetime_available l ON l.accommodation_id = d.accommodation_id
    )

SELECT  a.accommodation_id,
        a.adr,
        n.cost_per_night
FROM   adr_last_3_months      a
JOIN   normalised_cost        n USING (accommodation_id)
WHERE  a.adr < n.cost_per_night
ORDER BY a.accommodation_id;
```

**Execution outcome:**

| Estado | Columnas | Filas devueltas |
|---|---|---|
| Success | `['accommodation_id', 'adr', 'cost_per_night']` | [(31, Decimal('0E-20'), Decimal('0.81976793248945147679')), (44, Decimal('0E-20'), Decimal('0.15212814645308924485')), (47, Decimal('0E-20'), Decimal('0.72203562340966921120')), (48, Decimal('0E-20'), Decimal('0.78965986394557823129')), (57, Decimal('0E-20'), Decimal('0.34620758483033932136')), (114, Decimal('0E-20'), Decimal('0.56663755458515283843')), (129, Decimal('0E-20'), Decimal('0.07864806866952789700')), (162, Decimal('0E-20'), Decimal('0.37991525423728813559')), (163, Decimal('0E-20'), Decimal('0.16189189189189189189')), (183, Decimal('0E-20'), Decimal('0.36413301662707838480')), (218, Decimal('0E-20'), Decimal('0.69055803571428571429')), (270, Decimal('0E-20'), Decimal('0.73930555555555555556')), (273, Decimal('0E-20'), Decimal('0.15233532934131736527')), (350, Decimal('0E-20'), Decimal('0.09141987829614604462')), (358, Decimal('0E-20'), Decimal('0.68901075268817204301')), (385, Decimal('0E-20'), Decimal('0.85406315789473684211')), (387, Decimal('0E-20'), Decimal('0.84625806451612903226')), (393, Decimal('0E-20'), Decimal('0.73023305084745762712')), (415, Decimal('0E-20'), Decimal('0.59846960167714884696')), (416, Decimal('0E-20'), Decimal('0.15302083333333333333')), (419, Decimal('0E-20'), Decimal('0.36264192139737991266')), (422, Decimal('0E-20'), Decimal('0.79582608695652173913')), (440, Decimal('0E-20'), Decimal('0.86015695067264573991')), (453, Decimal('0E-20'), Decimal('0.78256410256410256410')), (476, Decimal('0E-20'), Decimal('0.19253863134657836645')), (526, Decimal('0.55967391304347826087'), Decimal('1.6476811594202899')), (560, Decimal('0E-20'), Decimal('0.23146881287726358149')), (578, Decimal('0E-20'), Decimal('1.2225854700854701')), (594, Decimal('0E-20'), Decimal('0.15488659793814432990'))]. |


---

## Question 5

**Question:** Which month had the most cancelled reservations, and what is the lost revenue for that month?

**Matched question_type:** cancellations_and_lost_revenue_by_month

**Context given to the model:**

```
Question type: cancellations_and_lost_revenue_by_month

Table: public_marts.fct_reservation
  - reservation_id (bigint)
  - accommodation_id (bigint)
  - start_date (date)
  - end_date (date)
  - nights (smallint)
  - status (text)
  - channel (text)
  - season (text)
  - total_price (numeric)
  - cancelled_at (timestamp with time zone)
  - booked_at (date)

Table: public_marts.mart_aggregates
  - accommodation_id (bigint)
  - month (date)
  - available_nights (integer)
  - revenue (numeric)
  - occupied_nights (bigint)
  - reservation_count (bigint)
  - cancelled_count (bigint)

Notes: "Lost revenue" is not a precomputed KPI in any mart -- compute it directly: SUM(total_price) FROM fct_reservation WHERE status = 'cancelled', grouped by month of start_date. total_price is populated regardless of status, so this works without reconstructing the price by hand -- the usual caveat (exclude cancelled from revenue) runs the OPPOSITE direction here: this question wants exactly the cancelled rows. Cancellation counts per month are also available pre-aggregated in mart_aggregates.cancelled_count if only the count (not the revenue figure) is needed.
```

**Raw model response:**

````
```sql
WITH cancelled_rev AS (
    SELECT
        -- truncate the start date to the month for grouping
        DATE_TRUNC('month', start_date)::date AS month,
        SUM(total_price) AS lost_revenue
    FROM public_marts.fct_reservation
    WHERE status = 'cancelled'
    GROUP BY DATE_TRUNC('month', start_date)::date
)

SELECT
    cr.month,
    ma.cancelled_count,
    cr.lost_revenue
FROM cancelled_rev cr
JOIN public_marts.mart_aggregates ma
  ON cr.month = ma.month
ORDER BY ma.cancelled_count DESC, cr.lost_revenue DESC
LIMIT 1;
```

This query returns the month with the highest number of cancelled reservations, along with the count of cancelations in that month and the total lost revenue from those cancellations.
````

**Extracted SQL:**

```sql
WITH cancelled_rev AS (
    SELECT
        -- truncate the start date to the month for grouping
        DATE_TRUNC('month', start_date)::date AS month,
        SUM(total_price) AS lost_revenue
    FROM public_marts.fct_reservation
    WHERE status = 'cancelled'
    GROUP BY DATE_TRUNC('month', start_date)::date
)

SELECT
    cr.month,
    ma.cancelled_count,
    cr.lost_revenue
FROM cancelled_rev cr
JOIN public_marts.mart_aggregates ma
  ON cr.month = ma.month
ORDER BY ma.cancelled_count DESC, cr.lost_revenue DESC
LIMIT 1;
```

**Execution outcome:**

| Estado | Columnas | Filas devueltas |
|---|---|---|
| Success | `['month', 'cancelled_count', 'lost_revenue']` | [(datetime.date(2026, 4, 1), 2, Decimal('63002.25'))]. |


---

## Question 6

**Question:** How many accommodations does property manager 26 have?

**Matched question_type:** accommodation_count

**Context given to the model:**

```
Question type: accommodation_count

Table: public_marts.dim_accommodation
  - accommodation_id (bigint)
  - property_manager_id (bigint)
  - accommodation_type (text)
  - city (text)
  - country (text)
  - listed_at (date)
  - min_stay_nights (smallint)
  - max_stay_nights (smallint)
  - max_guests (smallint)
  - no_smoking (boolean)
  - total_cost (numeric)

Notes: Deliberately the simplest question type -- a plain COUNT(*) on dim_accommodation, no joins, no business logic. Included as a control case: an agent that can't get this right has a more basic problem than anything routing/scope can fix.
```

**Raw model response:**

````
```sql
SELECT COUNT(*) AS accommodation_count
FROM public_marts.dim_accommodation
WHERE property_manager_id = 26;
```
````

**Extracted SQL:**

```sql
SELECT COUNT(*) AS accommodation_count
FROM public_marts.dim_accommodation
WHERE property_manager_id = 26;
```

**Execution outcome:**

| Estado | Columnas | Filas devueltas |
|---|---|---|
| Success | `['accommodation_count']` | [(20,)]. |


---

## Resultados en tabla

| Pregunta | Columnas devueltas | Resultado visible |
|---|---|---|
| 1. RevPAN del manager 20 | `accommodation_id`, `total_revpan` | 20 filas; máximo: 220 → 1006.01 |
| 2. Cancelaciones del alojamiento 15 | `cancelled_reservations` | **1** |
| 3. Ocupación del manager 9 en agosto 2026 | `occupancy_rate_pct` | **11.13%** |
| 4. ADR menor que coste normalizado | `accommodation_id`, `adr`, `cost_per_night` | **29 filas**; ejemplos: 31 → 0.00 vs 0.8198; 526 → 0.5597 vs 1.6477 |
| 5. Mes con más cancelaciones | `month`, `cancelled_count`, `lost_revenue` | **2026-04-01 · 2 · 63,002.25** |
| 6. Alojamientos del manager 26 | `accommodation_count` | **20** |
