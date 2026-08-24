# Content experiment — CONTEXT_MODE=docs_short

Generated 2026-08-21T21:06:51.410317+00:00. Model: openai/gpt-oss-20b (via NVIDIA). Every question below gets the exact same context: the FULL public_marts schema, unfiltered by question -- no routing, no per-question scoping.

**Context given to the model for every question:**

```
Table: public_marts.mart_kpi
  - accommodation_id (bigint) -- The accommodation this row rolls up.
  - month (date) -- First day of the calendar month this row covers (a date, not a formatted string).
  - occupancy_rate_pct (numeric) -- 100 * occupied_nights / available_nights, from mart_aggregates.
  - adr (numeric) -- Average Daily Rate: revenue / occupied_nights.
  - revpan (numeric) -- Revenue Per Available Night: revenue / available_nights.
  - avg_length_of_stay_nights (numeric) -- Average nights per confirmed reservation.
  - avg_booking_window_days (numeric) -- Average number of days between when a guest booked a reservation and their check-in date (`start_date - booked_at`), for confirmed reservations starting in this month.
  - cancellation_rate_pct (numeric) -- 100 * cancelled_count / reservation_count, from mart_aggregates.

Table: public_marts.mart_aggregates
  - accommodation_id (bigint) -- The accommodation this row rolls up.
  - month (date) -- First day of the calendar month this row covers.
  - available_nights (integer) -- Total calendar capacity for this accommodation in this month, bounded below by listed_at.
  - revenue (numeric) -- Revenue prorated by night across month boundaries, same technique as occupied_nights (per-night rate x overlapping nights in this month).
  - occupied_nights (bigint) -- Confirmed nights booked in this month.
  - reservation_count (bigint) -- Count of all reservations (confirmed and cancelled) starting in this month.
  - cancelled_count (bigint) -- Subset of reservation_count with status = 'cancelled', same month attribution (start_date, not cancelled_at).

Table: public_marts.fct_reservation
  - reservation_id (bigint) -- Primary key.
  - accommodation_id (bigint) -- The accommodation booked.
  - start_date (date) -- First night of the stay.
  - end_date (date) -- Checkout date (exclusive).
  - nights (smallint) -- end_date - start_date.
  - status (text) -- confirmed or cancelled.
  - channel (text) -- AIRBNB, BOOKING, or EXPEDIA.
  - season (text) -- HIGH or LOW.
  - total_price (numeric) -- GENERATED ALWAYS, populated regardless of status -- a cancelled reservation still has a non-null total_price (needed for lost-revenue questions).
  - cancelled_at (timestamp with time zone) -- Null unless status = 'cancelled'.
  - booked_at (date) -- The business date this reservation was actually booked (not a technical timestamp) -- always <= start_date.

Table: public_marts.dim_accommodation
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
  - total_cost (numeric) -- SUM(amount) from stg_accommodation_cost, lifetime-to-date.

```

## Question 1

**Question:** What is the total RevPAN for property manager 20, broken down by each of their accommodations?

**Raw model response:**

```
```sql
-- Total RevPAN per accommodation for property manager 20
SELECT
    k.accommodation_id,
    SUM(k.revpan) AS total_revpan
FROM public_marts.mart_kpi          AS k
JOIN public_marts.dim_accommodation AS d
  ON d.accommodation_id = k.accommodation_id
WHERE d.property_manager_id = 20
GROUP BY k.accommodation_id
ORDER BY total_revpan DESC;
```
```

**Extracted SQL:** `-- Total RevPAN per accommodation for property manager 20
SELECT
    k.accommodation_id,
    SUM(k.revpan) AS total_revpan
FROM public_marts.mart_kpi          AS k
JOIN public_marts.dim_accommodation AS d
  ON d.accommodation_id = k.accommodation_id
WHERE d.property_manager_id = 20
GROUP BY k.accommodation_id
ORDER BY total_revpan DESC;`

**Execution outcome:** Success. Columns: ['accommodation_id', 'total_revpan']. Rows (20, capped at 500): [(220, Decimal('1006.01')), (227, Decimal('730.48')), (230, Decimal('687.94')), (233, Decimal('642.96')), (222, Decimal('560.65')), (235, Decimal('495.37')), (228, Decimal('491.04')), (236, Decimal('449.19')), (229, Decimal('423.37')), (231, Decimal('422.63')), (221, Decimal('402.77')), (237, Decimal('304.68')), (224, Decimal('303.53')), (218, Decimal('297.50')), (225, Decimal('244.61')), (232, Decimal('224.67')), (223, Decimal('215.00')), (219, Decimal('186.07')), (226, Decimal('134.66')), (234, Decimal('91.38'))].

---

## Question 2

**Question:** How many cancelled reservations has accommodation 15 had?

**Raw model response:**

```
```sql
SELECT COUNT(*) AS cancelled_count
FROM public_marts.fct_reservation
WHERE accommodation_id = 15
  AND status = 'cancelled';
```
```

**Extracted SQL:** `SELECT COUNT(*) AS cancelled_count
FROM public_marts.fct_reservation
WHERE accommodation_id = 15
  AND status = 'cancelled';`

**Execution outcome:** Success. Columns: ['cancelled_count']. Rows (1, capped at 500): [(1,)].

---

## Question 3

**Question:** What is the occupancy rate for property manager 9 in August 2026?

**Raw model response:**

```
```sql
SELECT
    ROUND(
        100.0
        * SUM(a.occupied_nights)::numeric
        / NULLIF(SUM(a.available_nights), 0), 2
    ) AS occupancy_rate_pct
FROM public_marts.mart_aggregates   AS a
JOIN public_marts.dim_accommodation   AS d
    ON a.accommodation_id = d.accommodation_id
WHERE d.property_manager_id = 9
  AND a.month = DATE '2026-08-01';
```

This query:

1. **Filters** to the August 2026 month (`month = DATE '2026-08-01'`).
2. **Joins** to `dim_accommodation` to keep only accommodations managed by property manager 9.
3. **Calculates** the manager‑wide occupancy rate as the weighted average of occupied nights over available nights.  
   The result is expressed as a percentage rounded to two decimal places.
```

**Extracted SQL:** `SELECT
    ROUND(
        100.0
        * SUM(a.occupied_nights)::numeric
        / NULLIF(SUM(a.available_nights), 0), 2
    ) AS occupancy_rate_pct
FROM public_marts.mart_aggregates   AS a
JOIN public_marts.dim_accommodation   AS d
    ON a.accommodation_id = d.accommodation_id
WHERE d.property_manager_id = 9
  AND a.month = DATE '2026-08-01';`

**Execution outcome:** Success. Columns: ['occupancy_rate_pct']. Rows (1, capped at 500): [(Decimal('11.13'),)].

---

## Question 4

**Question:** Which accommodations have an ADR lower than their normalized daily cost over the last 3 months?

**Raw model response:**

```
Here’s a single read‑only query that pulls the data you asked for.

```sql
WITH last3 AS (
    /* 1️⃣  ADR and capacity for the past three calendar months  */
    SELECT
        km.accommodation_id,
        AVG(km.adr)                    AS avg_adr,                 -- mean ADR over last 3 mths
        SUM(ma.available_nights)      AS total_available        -- nights that could have been sold
    FROM public_marts.mart_kpi   AS km
    JOIN public_marts.mart_aggregates AS ma
        ON km.accommodation_id = ma.accommodation_id
       AND km.month = ma.month
    WHERE km.month >= date_trunc('month', current_date) - INTERVAL '2 month'
      AND km.month <  date_trunc('month', current_date) + INTERVAL '1 month'
    GROUP BY km.accommodation_id
),
daily_cost AS (
    /* 2️⃣  Normalised daily cost = lifetime total_cost ÷  available nights
            for the same 3‑month window. */
    SELECT
        a.accommodation_id,
        a.total_cost,
        l.total_available
    FROM public_marts.dim_accommodation AS a
    JOIN last3  AS l
        ON a.accommodation_id = l.accommodation_id
)
SELECT
    d.accommodation_id,
    d.total_cost                     AS lifetime_total_cost,
    l.total_available                 AS available_nights_last_3_mth,
    d.total_cost / NULLIF(l.total_available,0)  AS normalised_daily_cost,
    l.avg_adr                         AS avg_adr_last_3_mth
FROM daily_cost      AS d
JOIN last3            AS l
      ON d.accommodation_id = l.accommodation_id
WHERE d.total_cost / NULLIF(l.total_available,0) > l.avg_adr      -- ADR < cost
ORDER BY l.avg_adr;
```

### What the query does

1. **`last3`**  
   * Computes the average ADR (`avg_adr`) and sums the `available_nights` for each accommodation over the last three calendar months.

2. **`daily_cost`**  
   * Joins the above with `dim_accommodation` to pull in the lifetime‑to‑date `total_cost` and keep the total number of available nights (`total_available`) from the same 3‑month window.

3. **Final SELECT**  
   * Calculates the *normalised* daily cost as  
     `total_cost / total_available`.  
   * Keeps only those accommodations where this cost exceeds the average ADR.

Feel free to tweak the `ORDER BY` or add the manager/city columns if you’d like more context.
```

**Extracted SQL:** `WITH last3 AS (
    /* 1️⃣  ADR and capacity for the past three calendar months  */
    SELECT
        km.accommodation_id,
        AVG(km.adr)                    AS avg_adr,                 -- mean ADR over last 3 mths
        SUM(ma.available_nights)      AS total_available        -- nights that could have been sold
    FROM public_marts.mart_kpi   AS km
    JOIN public_marts.mart_aggregates AS ma
        ON km.accommodation_id = ma.accommodation_id
       AND km.month = ma.month
    WHERE km.month >= date_trunc('month', current_date) - INTERVAL '2 month'
      AND km.month <  date_trunc('month', current_date) + INTERVAL '1 month'
    GROUP BY km.accommodation_id
),
daily_cost AS (
    /* 2️⃣  Normalised daily cost = lifetime total_cost ÷  available nights
            for the same 3‑month window. */
    SELECT
        a.accommodation_id,
        a.total_cost,
        l.total_available
    FROM public_marts.dim_accommodation AS a
    JOIN last3  AS l
        ON a.accommodation_id = l.accommodation_id
)
SELECT
    d.accommodation_id,
    d.total_cost                     AS lifetime_total_cost,
    l.total_available                 AS available_nights_last_3_mth,
    d.total_cost / NULLIF(l.total_available,0)  AS normalised_daily_cost,
    l.avg_adr                         AS avg_adr_last_3_mth
FROM daily_cost      AS d
JOIN last3            AS l
      ON d.accommodation_id = l.accommodation_id
WHERE d.total_cost / NULLIF(l.total_available,0) > l.avg_adr      -- ADR < cost
ORDER BY l.avg_adr;`

**Execution outcome:** Success. Columns: ['accommodation_id', 'lifetime_total_cost', 'available_nights_last_3_mth', 'normalised_daily_cost', 'avg_adr_last_3_mth']. Rows (0, capped at 500): [].

---

## Question 5

**Question:** Which month had the most cancelled reservations, and what is the lost revenue for that month?

**Raw model response:**

```
```sql
-- Month with the highest number of cancelled reservations (and its lost revenue)
WITH cancelled AS (
    SELECT
        date_trunc('month', start_date)::date      AS month_start,
        COUNT(*)                                   AS cancelled_cnt,
        SUM(total_price)                           AS lost_rev
    FROM public_marts.fct_reservation
    WHERE status = 'cancelled'
    GROUP BY 1
)
SELECT
    month_start,
    cancelled_cnt,
    lost_rev
FROM cancelled
ORDER BY cancelled_cnt DESC, lost_rev DESC
LIMIT 1;
```

**What this query does**

1. **`date_trunc('month', start_date)`** — pulls the first day of the month for each cancelled reservation.
2. **Grouping** — aggregates by that month to count cancelled reservations and sum the `total_price` of those cancelled bookings.
3. **Ordering & limit** — picks the month with the highest count (and, if needed, the greatest lost revenue as a tie‑breaker).

*Result:*  
- `month_start` – the first day of the month that had the most cancellations.  
- `cancelled_cnt` – the number of cancelled reservations in that month.  
- `lost_rev` – the total lost revenue (cancellation loss) for that month.
```

**Extracted SQL:** `-- Month with the highest number of cancelled reservations (and its lost revenue)
WITH cancelled AS (
    SELECT
        date_trunc('month', start_date)::date      AS month_start,
        COUNT(*)                                   AS cancelled_cnt,
        SUM(total_price)                           AS lost_rev
    FROM public_marts.fct_reservation
    WHERE status = 'cancelled'
    GROUP BY 1
)
SELECT
    month_start,
    cancelled_cnt,
    lost_rev
FROM cancelled
ORDER BY cancelled_cnt DESC, lost_rev DESC
LIMIT 1;`

**Execution outcome:** Success. Columns: ['month_start', 'cancelled_cnt', 'lost_rev']. Rows (1, capped at 500): [(datetime.date(2026, 1, 1), 45, Decimal('61605.58'))].

---

## Question 6

**Question:** How many accommodations does property manager 26 have?

**Raw model response:**

```
```sql
SELECT COUNT(*) AS num_accommodations
FROM public_marts.dim_accommodation
WHERE property_manager_id = 26;
```
```

**Extracted SQL:** `SELECT COUNT(*) AS num_accommodations
FROM public_marts.dim_accommodation
WHERE property_manager_id = 26;`

**Execution outcome:** Success. Columns: ['num_accommodations']. Rows (1, capped at 500): [(20,)].

---
