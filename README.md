# Toronto Short-Term Rental Market Intelligence

An end-to-end data analytics project analysing Toronto's short-term rental market using Inside Airbnb data. The goal was to generate market intelligence for a property management startup evaluating entry into Toronto's STR market.

**Tools used:** Google BigQuery (SQL) · Power BI

## Data Source
[Inside Airbnb](http://insideairbnb.com/) — Creative Commons Attribution 4.0 International License.

---

## About This Project

End-to-end solo project covering data profiling, SQL analysis in Google BigQuery, and dashboard design in Power BI. SQL queries were written to answer four specific business questions, imported directly into Power BI via the Google BigQuery connector, and used to build a two-page stakeholder report.

---

## Business Problem

A property management startup is considering expanding into Toronto's short-term rental market and needs data-driven answers to four key questions before committing resources:

1. **Demand signals** — Where is demand strongest?
2. **Supply distribution** — Where is inventory concentrated and what types dominate?
3. **Competitive landscape** — Professional operators vs casual hosts — who controls the market?
4. **Licensing and compliance** — What is the regulatory risk and where is it concentrated?

---

## Approach

The analysis was structured around four business questions. For each question, the relevant metric was identified, aggregated in BigQuery using SQL, and visualised in Power BI for stakeholder presentation.

- **Demand** — measured by booking activity (`number_of_reviews_ltm`) rather than listing count. A neighbourhood with many listings but few recent bookings is not a strong demand signal. Reviews per month was used as a secondary metric to measure booking velocity per listing, surfacing high-demand pockets that are currently undersupplied
- **Supply** — measured by listing count and room type mix to understand what inventory exists, where it is concentrated, and which product type dominates the market
- **Competition** — hosts were segmented by portfolio size into casual (1 listing) and professional (2+ listings) tiers to understand who actually controls the market vs who is a passive or dormant participant
- **Compliance** — null values in the `license` column were treated as unlicensed. Compliance was then cross-referenced with host tier and geography to identify where regulatory risk is concentrated

---

## SQL Queries

### Q1. Where is supply concentrated?

**Top 15 neighbourhoods by listing count**
```sql
SELECT
  neighbourhood,
  COUNT(*) AS listing_count,
  ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM `<project>.<dataset>.listings`), 1) AS pct_of_total
FROM `<project>.<dataset>.listings`
GROUP BY neighbourhood
ORDER BY listing_count DESC
LIMIT 15;
```

**Top 15 neighbourhoods by listing count and room type**
```sql
SELECT
  neighbourhood,
  room_type,
  COUNT(*) AS listing_count
FROM `<project>.<dataset>.listings`
WHERE neighbourhood IN (
  SELECT neighbourhood
  FROM `<project>.<dataset>.listings`
  GROUP BY neighbourhood
  ORDER BY COUNT(*) DESC
  LIMIT 15
)
GROUP BY neighbourhood, room_type
ORDER BY listing_count DESC;
```

**Room type distribution — citywide**
```sql
SELECT
  room_type,
  COUNT(*) AS listing_count
FROM `<project>.<dataset>.listings`
GROUP BY room_type
ORDER BY listing_count DESC;
```

---

### Q2. Where is demand strongest?

```sql
SELECT
  neighbourhood,
  COUNT(*) AS listing_count,
  ROUND(AVG(COALESCE(SAFE_CAST(reviews_per_month AS FLOAT64), 0)), 2) AS avg_rpm,
  APPROX_QUANTILES(COALESCE(SAFE_CAST(reviews_per_month AS FLOAT64), 0), 2)[OFFSET(1)] AS median_rpm,
  SUM(SAFE_CAST(number_of_reviews AS INT64)) AS total_reviews
FROM `<project>.<dataset>.listings`
WHERE SAFE_CAST(number_of_reviews AS INT64) > 0
GROUP BY neighbourhood
HAVING COUNT(*) >= 10
ORDER BY avg_rpm DESC
LIMIT 15;
```

---

### Q3. Professional operators vs casual hosts

```sql
SELECT
  CASE
    WHEN calculated_host_listings_count > 1 THEN 'Professional'
    ELSE 'Casual'
  END AS host_type,
  COUNT(*) AS listing_count,
  ROUND(AVG(COALESCE(SAFE_CAST(reviews_per_month AS FLOAT64), 0)), 2) AS avg_rpm,
  APPROX_QUANTILES(COALESCE(SAFE_CAST(reviews_per_month AS FLOAT64), 0), 2)[OFFSET(1)] AS median_rpm,
  SUM(SAFE_CAST(number_of_reviews AS INT64)) AS total_reviews,
  ROUND(AVG(availability_365)) AS avg_availability
FROM `<project>.<dataset>.listings`
WHERE SAFE_CAST(number_of_reviews AS INT64) > 0
GROUP BY host_type
HAVING COUNT(*) >= 10
ORDER BY avg_rpm DESC;
```

---

### Q4. Licensing compliance

**Overall licensing rate**
```sql
SELECT
  CASE WHEN license IS NULL THEN 'No' ELSE 'Yes' END AS licensed_flag,
  COUNT(*) AS number_of_listings,
  ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM `<project>.<dataset>.listings`), 1) AS pct_of_total
FROM `<project>.<dataset>.listings`
GROUP BY licensed_flag
ORDER BY number_of_listings DESC;
```

**Licensing by neighbourhood**
```sql
SELECT
  neighbourhood,
  COUNT(CASE WHEN license IS NULL THEN id ELSE NULL END)     AS unlicensed,
  COUNT(CASE WHEN license IS NOT NULL THEN id ELSE NULL END) AS licensed,
  ROUND(COUNT(CASE WHEN license IS NOT NULL THEN id ELSE NULL END) * 1.0
    / COUNT(*), 2) AS licensed_pct
FROM `<project>.<dataset>.listings`
GROUP BY neighbourhood
ORDER BY licensed DESC;
```

---

### KPI Cards — Page 1 (Supply & Demand)

```sql
SELECT
  COUNT(*)                                                      AS total_listings,
  COUNT(DISTINCT neighbourhood)                                  AS total_neighbourhoods,
  (
    SELECT COUNT(*)
    FROM `<project>.<dataset>.listings`
    GROUP BY neighbourhood
    ORDER BY COUNT(*) DESC
    LIMIT 1
  )                                                             AS top_neighbourhood_count,
  CONCAT(CAST(ROUND(100.0 * COUNTIF(room_type = 'Entire home/apt')
    / COUNT(*), 1) AS STRING), '%')                             AS entire_home_pct,
  ROUND(100.0 * (
    SELECT SUM(total_ltm) FROM (
      SELECT SUM(SAFE_CAST(number_of_reviews_ltm AS INT64)) AS total_ltm
      FROM `<project>.<dataset>.listings`
      GROUP BY neighbourhood
      ORDER BY total_ltm DESC
      LIMIT 10
    )
  ) / SUM(SAFE_CAST(number_of_reviews_ltm AS INT64)), 1)        AS top10_demand_share_pct
FROM `<project>.<dataset>.listings`
```

---

### KPI Cards — Page 2 (Competition & Compliance)

```sql
SELECT
  COUNT(DISTINCT host_id)                                        AS total_hosts,
  CONCAT(CAST(ROUND(100.0 * COUNTIF(calculated_host_listings_count = 1)
    / COUNT(DISTINCT host_id), 1) AS STRING), '%')              AS casual_host_pct,
  CONCAT(CAST(ROUND(100.0 * COUNTIF(license IS NULL)
    / COUNT(*), 1) AS STRING), '%')                             AS unlicensed_pct,
  COUNT(DISTINCT CASE WHEN calculated_host_listings_count > 5
    THEN host_id END)                                           AS professional_hosts
FROM `<project>.<dataset>.listings`
```

---

## Dashboard Preview

The SQL queries above were run directly in Power BI's Power Query editor using the Google BigQuery connector. Each query was loaded as a separate table and used to build the report — no CSV exports or intermediate files were needed.

**Page 1 — Supply & Demand by Neighbourhood**

![Supply and Demand Dashboard](images/page1.png)

**Page 2 — Competition & Compliance**

![Competition and Compliance Dashboard](images/page2.png)

---

## Key Findings

### Strategic Summary

| Business Question | Finding | Implication |
|---|---|---|
| Where is demand? | High-demand, low-supply outer neighbourhoods | Enter here first, not Waterfront |
| Where is supply? | Concentrated in central Toronto, entire homes dominate | Entire home portfolio is the right product |
| Who are competitors? | Mostly casual hosts, few active professionals | Low barrier to becoming a top-tier operator |
| What is the regulatory risk? | 43.4% of market is unlicensed | Being licensed is an advantage, not just compliance |

---

### Market Overview
- Toronto's STR market has **15,704 active listings** across **140 neighbourhoods**
- **67.1% of all listings are entire home/apartment** — the highest-value room type for a property management operator

---

### 1. Where is demand strongest?
- **Demand is heavily concentrated in central Toronto** — the top 10 neighbourhoods account for **49.5% of all booking activity** despite there being 140 neighbourhoods total
- **Waterfront Communities-The Island is the dominant market** — it drives ~23% of all citywide bookings and holds 2,653 listings, nearly 4× the next largest neighbourhood (Niagara at 583)
- **The highest-opportunity entry markets are not the biggest ones** — Kingsview Village-The Westway, Regent Park, Wychwood, Playter Estates-Danforth and Blake-Jones show above-average booking velocity (2.1–2.2 reviews/month) but very few listings — strong demand that existing supply has not yet met
- **Recommendation:** prioritise high-demand, low-supply neighbourhoods — these offer the best return with the least competition

---

### 2. Supply distribution — where is inventory concentrated?
- **Supply is top-heavy and skewed towards central Toronto** — the top 15 neighbourhoods hold a disproportionate share of all listings, with Waterfront Communities alone representing ~17% of citywide supply
- **Entire home listings dominate at 67.1%** — private rooms make up 32% but show nearly identical booking velocity, making them a lower-priority segment for a new operator
- **Outer Toronto neighbourhoods are significantly undersupplied** relative to the demand signals they show — a new operator can capture market share here with less head-to-head competition

---

### 3. Competitive landscape — professional vs casual hosts
- **The market is fragmented** — 80% of the 10,398 total hosts hold just one listing; most of the market is individual casual hosts rather than organised operators
- **Professional operators are few but impactful** — only 222 hosts (2.1%) run 6 or more listings, yet collectively control ~16% of total supply
- **Many large-portfolio operators are not active competitors** — the biggest operators show near-zero recent booking activity and no license, pointing to dormant or speculative listings rather than genuine competition
- **Recommendation:** a professionally managed, well-reviewed, licensed portfolio of 10–20 properties would place a new entrant in the top tier of operators almost immediately

---

### 4. Licensing and compliance — regulatory risk and opportunity
- **43.4% of all listings (6,843) operate without a valid license** — a significant share of current supply is non-compliant with Toronto's STR licensing requirements
- **Non-compliance grows with portfolio size** — casual hosts (1 listing) show 69% compliance; the largest operators (21+ listings) show only 13.5% compliance, meaning the biggest players carry the highest regulatory risk
- **Highest compliance is in outer neighbourhoods** — Humber Summit (83%), Beechborough-Greenbrook (81%) and Thistletown-Beaumond Heights (81%) lead; the downtown core where demand is highest has the most non-compliant supply
- **Entering fully licensed is a strategic advantage** — if Toronto increases STR enforcement, unlicensed competitors face removal from the platform while a compliant operator captures their demand
- **Recommendation:** full licensing compliance should be non-negotiable — it is both a risk management position and a competitive differentiator in a market where 43.4% of competitors are vulnerable to enforcement action

---

## Data Notes

- **Price column is fully null** in this extract — revenue-based analysis is not possible from this dataset alone
- **`neighbourhood_group` is fully null** — all geographic analysis is at the `neighbourhood` level (140 distinct)
- **`reviews_per_month`** is a lifetime average, not a current rate — `number_of_reviews_ltm` is used as the primary demand proxy
- **`calculated_host_listings_count`** matches actual row counts in this extract and is reliable for host tier classification
- BigQuery total (15,704) differs from raw CSV (15,776) by ~72 rows — the raw CSV contains broken rows caused by bad quoting in the `name` column; BigQuery skips these on import rather than failing

---

## Repository Contents

| File | Description |
|---|---|
| `toronto_str_market.pbix` | Interactive 2-page Power BI dashboard |
| `Toronto_STR_Market_Intelligence.pptx` | 2-page presentation with dashboard screenshots and key findings per business question |

> SQL queries are documented directly in this README. All queries were written and executed in Google BigQuery — no local `.sql` files were used.

---

## Connect

[LinkedIn](http://www.linkedin.com/in/shobhitasohal)
