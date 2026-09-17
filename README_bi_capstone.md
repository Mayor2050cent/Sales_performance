# Sales Performance Dashboard — BI Capstone Project

**A star-schema BI data model with a live, reusable Power BI reporting dashboard.**

## Business Question

Build an ongoing sales performance reporting system — not a one-time
analysis — covering revenue trends, growth, profitability, and top
customers, structured the way a real BI tool is built rather than a
single static report.

## Dataset

Sample Superstore Dataset (Kaggle) — 9,994 orders spanning 2014–2017.
This project reuses the same raw data as an earlier flat-table sales
analysis, but rebuilds it from the ground up as a proper BI data model.

## What Makes This Different From a Standard Data-Analysis Project

- **Star schema**, not a flat file — one fact table (`fact_orders`)
  connected to four dimension tables (`dim_customer`, `dim_product`,
  `dim_region`, `dim_date`)
- **Native DAX measures** for time intelligence and filter context
  (`SAMEPERIODLASTYEAR`, `CALCULATE`, `FILTER`, `DIVIDE`) instead of
  pre-calculated values imported from Python
- **Power Query transformation steps** that are saved and automatically
  re-applied on every data refresh, rather than one-time manual edits
- **Framed as an ongoing reporting tool** — a KPI-card-led dashboard
  layout, the way a real company's sales dashboard would look

## Process

1. **Design** — Mapped the flat Superstore table to a star schema:
   fact table for measures/keys, dimension tables for descriptive
   attributes
2. **Build** — Created and populated all 5 tables in MySQL; caught and
   fixed a real duplicate-key bug during the build (see below)
3. **Model** — Loaded all 5 tables into Power BI and built the
   relationships (fact table connected to all 4 dimensions)
4. **Measure** — Built 8 DAX measures, including time-intelligence and
   filter-context calculations
5. **Transform** — Used Power Query to clean data and add calculated/
   conditional columns as saved, repeatable steps
6. **Report** — Built a KPI-card-led dashboard: top-line metrics, YoY
   growth table, cumulative trend chart, top customers

## Real Issues Solved Along the Way

- **Duplicate product IDs**: some products had slightly different name/
  category text across orders. Deduplicating on all columns instead of
  just `product_id` inflated the fact table from 9,994 to 10,331 rows.
  Fixed by grouping on `product_id` alone.
- **Broken time intelligence**: an initial `dim_date` table only
  contained dates that appeared in actual orders (with gaps). This
  silently broke `SAMEPERIODLASTYEAR` and running-total measures, which
  only returned values in the grand-total row. Fixed by rebuilding
  `dim_date` as a complete, gap-free calendar table — a key lesson that
  date dimensions need every date in range, not just dates present in
  the data.

## Dashboard

![BI Capstone Dashboard](dashboard_screenshot.png)

🔗 [View live interactive dashboard](PASTE_YOUR_PUBLISHED_LINK_HERE)

## Key Findings

- **Gross margin: 12.47%** — thin for retail, driven largely by the
  Furniture category (high revenue, near-zero profit)
- **Revenue growth**: $484,248 (2014) → $733,215 (2017), ~15.7% average
  year-over-year growth
- **Top customer**: Sean Miller, $25,043 lifetime value — independently
  confirmed via both a direct SQL ranking query and the Power BI
  dashboard's Top 10 Customers visual

## Tools Used

- **MySQL** — star schema design and build
- **Power BI** — data modeling, relationships, DAX, Power Query,
  dashboard design

## Files in This Repo

- `bi_capstone_scripts.md` — full SQL (star schema), DAX measures, and
  Power Query documentation
- `dashboard_screenshot.png` — static preview of the Power BI dashboard
