# Chapter 13 — Transcript: the live agent

Generated 2026-08-21T21:13:34.715043+00:00. Model: openai/gpt-oss-20b (via NVIDIA). Unlike chapter 07, each question is routed to a matched question_type from agent_routing.yml and given a scoped schema context before the model sees it. Same 6 questions as chapter 07's transcript, for a direct before/after comparison.

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
**SQL**

```sql
SELECT  d.accommodation_id,
        d.accommodation_type,
        d.city,
        d.country,
        SUM(k.revpand) AS total_revpan
FROM   public_marts.mart_kpi        AS k
JOIN   public_marts.dim_accommodation AS d
       ON k.accommodation_id = d.accommodation_id
WHERE  d.property_manager_id = 20
GROUP BY d.accommodation_id,
         d.accommodation_type,
         d.city,
         d.country
ORDER BY total_revpan DESC;
```

This query aggregates the **RevPAN** (`revpan` in `mart_kpi`) for every accommodation owned by property manager **20**, returning the total revenue per accommodation (optionally including its type, city, and country).
````

**Extracted SQL:**

```sql
SELECT  d.accommodation_id,
        d.accommodation_type,
        d.city,
        d.country,
        SUM(k.revpand) AS total_revpan
FROM   public_marts.mart_kpi        AS k
JOIN   public_marts.dim_accommodation AS d
       ON k.accommodation_id = d.accommodation_id
WHERE  d.property_manager_id = 20
GROUP BY d.accommodation_id,
         d.accommodation_type,
         d.city,
         d.country
ORDER BY total_revpan DESC;
```

**Execution outcome:**

| Estado | Detalle |
|---|---|
| Error | Postgres error: column k.revpand does not exist LINE 5:         SUM(k.revpand) AS total_revpan ^ HINT:  Perhaps you meant to reference the column "k.revpan". |



---

## Question 2

**Question:** How many cancelled reservations has accommodation 15 had?

**Matched question_type:** (none -- fallback)

**Extracted SQL:**

```sql
-- No SQL extracted from the model response.
```

**Execution outcome:**

| Estado | Detalle |
|---|---|
| Fallback | I don't have a defined routing rule for this question. Can you rephrase it in terms of occupancy, revenue, ADR/RevPAN, cancellations, or accommodation counts -- or tell me which table/mart you'd expect this to come from? |


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
-- Occupancy rate for property manager 9 in August 2026
SELECT
    AVG(k.occupancy_rate_pct) AS occupancy_rate_pct
FROM
    public_marts.mart_kpi k
JOIN
    public_marts.dim_accommodation d
    ON k.accommodation_id = d.accommodation_id
WHERE
    d.property_manager_id = 9
    AND k.month >= DATE '2026-08-01'
    AND k.month <  DATE '2026-09-01';
```
````

**Extracted SQL:**

```sql
-- Occupancy rate for property manager 9 in August 2026
SELECT
    AVG(k.occupancy_rate_pct) AS occupancy_rate_pct
FROM
    public_marts.mart_kpi k
JOIN
    public_marts.dim_accommodation d
    ON k.accommodation_id = d.accommodation_id
WHERE
    d.property_manager_id = 9
    AND k.month >= DATE '2026-08-01'
    AND k.month <  DATE '2026-09-01';
```

**Execution outcome:**

| occupancy_rate_pct |
|---|
| 11.1300000000000000 |
---

## Question 4

**Question:** Which accommodations have an ADR lower than their normalized daily cost over the last 3 months?

**Matched question_type:** adr_revpan

**Context given to the model:**

```
Question type: adr_revpan

Table: public_marts.mart_kpi
  - accommodation_id (bigint)
  - month (date)
  - occupancy_rate_pct (numeric)
  - adr (numeric)
  - revpan (numeric)
  - avg_length_of_stay_nights (numeric)
  - avg_booking_window_days (numeric)
  - cancellation_rate_pct (numeric)

Notes: Direct read from mart_kpi. No joins needed. Use this for a plain ADR/RevPAN figure with no property-manager scoping and no cost comparison -- see revenue_per_property_manager and adr_vs_cost for those more specific cases.
```

**Raw model response:**

```
I’m sorry, but the only table I have access to (`public_marts.mart_kpi`) does not contain any column that represents a *normalized daily cost* (such as `"normalized_daily_cost"` or a similar field). Without that information I can’t compare it to the ADR in a SQL query.

If you have another table that holds the daily cost per accommodation and you can share its schema, I can write a query that joins to it to return the accommodations whose ADR is lower than the normalized daily cost.
```

**Extracted SQL:**

```sql
-- No SQL extracted from the model response.
```

**Execution outcome:**

| Estado | Detalle |
|---|---|
| Sin SQL | No SQL extracted from the model's response. |


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
-- Month with the highest number of cancelled reservations
-- and the associated lost revenue
WITH cancelled_counts AS (
    SELECT
        month,
        cancelled_count
    FROM public_marts.mart_aggregates
),
lost_rev_by_month AS (
    SELECT
        date_trunc('month', start_date)::date      AS month,
        SUM(total_price)                          AS lost_revenue
    FROM public_marts.fct_reservation
    WHERE status = 'cancelled'
    GROUP BY month
)
SELECT
    cc.month,
    cc.cancelled_count,
    lr.lost_revenue
FROM cancelled_counts cc
JOIN lost_rev_by_month lr ON cc.month = lr.month
WHERE cc.cancelled_count = (
    SELECT MAX(cancelled_count) FROM cancelled_counts
);
```

This query:

1. Pulls the monthly `cancelled_count` from `mart_aggregates`.
2. Computes the total price of cancelled reservations per month from `fct_reservation`.
3. Matches the two month columns.
4. Returns the month that had the highest number of cancelled reservations along with that month's cancelled count and the corresponding lost revenue.
````

**Extracted SQL:**

```sql
-- Month with the highest number of cancelled reservations
-- and the associated lost revenue
WITH cancelled_counts AS (
    SELECT
        month,
        cancelled_count
    FROM public_marts.mart_aggregates
),
lost_rev_by_month AS (
    SELECT
        date_trunc('month', start_date)::date      AS month,
        SUM(total_price)                          AS lost_revenue
    FROM public_marts.fct_reservation
    WHERE status = 'cancelled'
    GROUP BY month
)
SELECT
    cc.month,
    cc.cancelled_count,
    lr.lost_revenue
FROM cancelled_counts cc
JOIN lost_rev_by_month lr ON cc.month = lr.month
WHERE cc.cancelled_count = (
    SELECT MAX(cancelled_count) FROM cancelled_counts
);
```

**Execution outcome:**

| month | cancelled_count | lost_revenue |
|---|---|---|
| 2026-02-01 | 2 | 62284.22 |
| 2026-01-01 | 2 | 61605.58 |
| 2026-07-01 | 2 | 53587.96 |
| 2026-02-01 | 2 | 62284.22 |
| 2026-05-01 | 2 | 58223.90 |
| 2026-08-01 | 2 | 44628.81 |
| 2025-10-01 | 2 | 46164.30 |
| 2025-09-01 | 2 | 51502.68 |
| 2026-03-01 | 2 | 57924.50 |
| 2025-11-01 | 2 | 49429.03 |
| 2026-07-01 | 2 | 53587.96 |
| 2026-03-01 | 2 | 57924.50 |
| 2026-03-01 | 2 | 57924.50 |
| 2026-01-01 | 2 | 61605.58 |
| 2026-04-01 | 2 | 63002.25 |
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

| accommodation_count |
|---|
| 20 |
---

## Resultados en tabla

| Pregunta | Estado | Resultado visible |
|---|---|---|
| 1. RevPAN del manager 20 | Error SQL | `column k.revpand does not exist` |
| 2. Cancelaciones del alojamiento 15 | Fallback | No había regla de routing |
| 3. Ocupación del manager 9 en agosto 2026 | Éxito | **11.13%** |
| 4. ADR menor que coste normalizado | Sin SQL | No se produjo una consulta ejecutable |
| 5. Mes con más cancelaciones | Fan-out | 15 filas; ejemplo: 2026-04-01 → 2 · 63,002.25 |
| 6. Alojamientos del manager 26 | Éxito | **20** |
