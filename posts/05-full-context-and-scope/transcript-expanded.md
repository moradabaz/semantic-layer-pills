# Post 5 follow-up — 6 new questions, robustness check

Generated 2026-08-21T21:40:28.727875+00:00. Model: openai/gpt-oss-20b (via NVIDIA). Same fully-fixed agent_routing.yml/agent_scope.yml as chapter 13's final transcript -- these 6 questions are new, chosen to exercise question_types the original 6 never touched.

## Question 1

**Question:** What is the average length of stay across all accommodations?

**Matched question_type:** (none -- fallback)

**Extracted SQL:** (none)

**Execution outcome:** I don't have a defined routing rule for this question. Can you rephrase it in terms of occupancy, revenue, ADR/RevPAN, cancellations, or accommodation counts -- or tell me which table/mart you'd expect this to come from?

---

## Question 2

**Question:** On average, how far in advance do guests book before their stay?

**Matched question_type:** (none -- fallback)

**Extracted SQL:** (none)

**Execution outcome:** I don't have a defined routing rule for this question. Can you rephrase it in terms of occupancy, revenue, ADR/RevPAN, cancellations, or accommodation counts -- or tell me which table/mart you'd expect this to come from?

---

## Question 3

**Question:** Which channel has the highest cancellation rate?

**Matched question_type:** cancellation_rate

**Context given to the model:**

```
Question type: cancellation_rate

Table: public_marts.mart_kpi
  - accommodation_id (bigint)
  - month (date)
  - occupancy_rate_pct (numeric)
  - adr (numeric)
  - revpan (numeric)
  - avg_length_of_stay_nights (numeric)
  - avg_booking_window_days (numeric)
  - cancellation_rate_pct (numeric)

Notes: Direct read from mart_kpi. Already correctly excludes revenue for cancelled reservations upstream -- no extra filtering needed here.
```

**Raw model response:**

```
I’m sorry, but the `public_marts.mart_kpi` table you can query does not contain any column that identifies a booking channel.  
Because of that, there’s no way to compute or compare cancellation rates by channel from the data available to me. If you have another table that includes channel information, I can combine it with `mart_kpi` to answer your question.
```

**Extracted SQL:** `with `mart_kpi` to answer your question.`

**Execution outcome:** Postgres error: syntax error at or near "`"
LINE 1: with `mart_kpi` to answer your question.
             ^


---

## Question 4

**Question:** Is occupancy higher during high season or low season?

**Matched question_type:** seasonality

**Context given to the model:**

```
Question type: seasonality

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

Notes: season is a reservation-level dimension, same reasoning as channel above -- not baked into mart_kpi/mart_aggregates, query fct_reservation directly (GROUP BY season, filter status = 'confirmed' for revenue questions). season is a frozen snapshot computed server-side at booking time from season_calendar, matched on (city, accommodation_type) -- if season_calendar changes later, existing reservations keep their original value (same snapshot caveat pattern as property_manager_id).
```

**Raw model response:**

````
```sql
-- Occupancy comparison (total nights booked) between High Season and Low Season
WITH confirmed AS (
    SELECT
        season,
        nights,
        COUNT(*) OVER (PARTITION BY season) AS n_reservations
    FROM public_marts.fct_reservation
    WHERE status = 'confirmed'
), season_totals AS (
    SELECT
        season,
        SUM(nights)          AS total_nights,
        SUM(n_reservations) AS total_reservations
    FROM confirmed
    GROUP BY season
)
SELECT
    st.season,
    st.total_nights,
    st.total_reservations,
    CASE
        WHEN SUM(st.total_nights) FILTER (WHERE season = 'High Season')
             > SUM(st.total_nights) FILTER (WHERE season = 'Low Season')
        THEN 'Higher during High Season'
        ELSE 'Higher during Low Season'
    END AS occupancy_comparison
FROM season_totals st
GROUP BY st.season, st.total_nights, st.total_reservations;
```

This query:

1. **Filters** for confirmed reservations only, because those represent actual occupancy.  
2. **Aggregates** total nights and number of reservations per season.  
3. **Compares** the aggregated values for “High Season” vs. “Low Season” and returns a label indicating which season has the higher summed occupancy (by total booked nights).  
4. The results also show the raw totals for each season, so you can see the numbers behind the comparison.
````

**Extracted SQL:** `-- Occupancy comparison (total nights booked) between High Season and Low Season
WITH confirmed AS (
    SELECT
        season,
        nights,
        COUNT(*) OVER (PARTITION BY season) AS n_reservations
    FROM public_marts.fct_reservation
    WHERE status = 'confirmed'
), season_totals AS (
    SELECT
        season,
        SUM(nights)          AS total_nights,
        SUM(n_reservations) AS total_reservations
    FROM confirmed
    GROUP BY season
)
SELECT
    st.season,
    st.total_nights,
    st.total_reservations,
    CASE
        WHEN SUM(st.total_nights) FILTER (WHERE season = 'High Season')
             > SUM(st.total_nights) FILTER (WHERE season = 'Low Season')
        THEN 'Higher during High Season'
        ELSE 'Higher during Low Season'
    END AS occupancy_comparison
FROM season_totals st
GROUP BY st.season, st.total_nights, st.total_reservations;`

**Execution outcome:** Success. Columns: ['season', 'total_nights', 'total_reservations', 'occupancy_comparison']. Rows (2, capped at 500): [('HIGH', 11068, Decimal('2592100'), 'Higher during Low Season'), ('LOW', 23171, Decimal('11417641'), 'Higher during Low Season')].

---

## Question 5

**Question:** Do stricter occupancy rules correlate with more cancellations?

**Matched question_type:** occupancy

**Context given to the model:**

```
Question type: occupancy

Table: public_marts.mart_kpi
  - accommodation_id (bigint)
  - month (date)
  - occupancy_rate_pct (numeric)
  - adr (numeric)
  - revpan (numeric)
  - avg_length_of_stay_nights (numeric)
  - avg_booking_window_days (numeric)
  - cancellation_rate_pct (numeric)

Notes: Direct read from mart_kpi. No joins needed.
```

**Raw model response:**

````
**Correlation between occupancy rate and cancellation rate**

```sql
--   1. Compute the mean occupancy and cancellation rates
--   2. Derive deviations from those means
--   3. Compute the Pearson correlation coefficient

WITH avgs AS (
    SELECT
        AVG(occupancy_rate_pct)   AS mean_occ,
        AVG(cancellation_rate_pct) AS mean_can
    FROM public_marts.mart_kpi
),
dev AS (
    SELECT
        occupancy_rate_pct,
        cancellation_rate_pct,
        (occupancy_rate_pct - avg.mean_occ) AS d_occ,
        (cancellation_rate_pct - avg.mean_can) AS d_can
    FROM public_marts.mart_kpi m
    CROSS JOIN avgs avg
)
SELECT
    SUM(d_occ * d_can) /
    NULLIF(SQRT(SUM(d_occ * d_occ) * SUM(d_can * d_can)), 0) AS correlation_coefficient
FROM dev;
```

The returned `correlation_coefficient` will indicate whether stricter occupancy rules (lower occupancy rate) tend to be associated with higher cancellation rates:

* **Positive value** – stricter rules tend to coincide with *more* cancellations.
* **Negative value** – stricter rules tend to coincide with *fewer* cancellations.
* **Near zero** – little or no linear relationship.
````

**Extracted SQL:** `--   1. Compute the mean occupancy and cancellation rates
--   2. Derive deviations from those means
--   3. Compute the Pearson correlation coefficient

WITH avgs AS (
    SELECT
        AVG(occupancy_rate_pct)   AS mean_occ,
        AVG(cancellation_rate_pct) AS mean_can
    FROM public_marts.mart_kpi
),
dev AS (
    SELECT
        occupancy_rate_pct,
        cancellation_rate_pct,
        (occupancy_rate_pct - avg.mean_occ) AS d_occ,
        (cancellation_rate_pct - avg.mean_can) AS d_can
    FROM public_marts.mart_kpi m
    CROSS JOIN avgs avg
)
SELECT
    SUM(d_occ * d_can) /
    NULLIF(SQRT(SUM(d_occ * d_occ) * SUM(d_can * d_can)), 0) AS correlation_coefficient
FROM dev;`

**Execution outcome:** Success. Columns: ['correlation_coefficient']. Rows (1, capped at 500): [(Decimal('-0.2623745766771795795089639914133612811330111788658210009401451497'),)].

---

## Question 6

**Question:** What is the capital of France?

**Matched question_type:** (none -- fallback)

**Extracted SQL:** (none)

**Execution outcome:** I don't have a defined routing rule for this question. Can you rephrase it in terms of occupancy, revenue, ADR/RevPAN, cancellations, or accommodation counts -- or tell me which table/mart you'd expect this to come from?

---

## Resultados en tabla

| Pregunta | Routing | Resultado visible |
|---|---|---|
| 1. Longitud media de estancia | Fallback | No hay regla de routing definida |
| 2. Antelación media de reserva | Fallback | No hay regla de routing definida |
| 3. Mayor tasa de cancelación por canal | `cancellation_rate` | No existe `channel` en la tabla; SQL inválido y error de sintaxis |
| 4. Ocupación por temporada | `seasonality` | HIGH: 11,068 noches y 2,592,100 reservas; LOW: 23,171 noches y 11,417,641 reservas; mayor en **Low Season** |
| 5. Reglas de ocupación vs cancelaciones | `occupancy` | Correlación: **-0.2623745766771795795** |
| 6. Capital de Francia | Fallback | No hay regla de routing definida |
