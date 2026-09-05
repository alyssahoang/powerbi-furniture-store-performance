# Maven Analytics portfolio entry — Furniture Sales Performance

Paste-ready copy for the three Maven fields. All figures verified against the source files.

---

## Project Title

**Furniture Sales Performance Analysis in Power BI**

Alternates:

- Furniture Sales Performance: Store, Product and Demand Analysis
- Three-Store Furniture Retailer: An 18-Month Sales Review

---

## Project Excerpt

> Project to practice end-to-end data modelling and dashboard design in Power BI, using 18 months of
> sales data from a three-store furniture retailer: 50,200 transactions, $8.78M in revenue, 80
> products.

---

## Project Description

Project to practice end-to-end data preparation, modelling and dashboard design in Power BI. The data
comes from a fictitious furniture retailer with three branches in Long Island City, Paramus and Red
Hook. Transactions were supplied as eighteen separate monthly Excel files covering January 2025 to
June 2026, alongside a product list and a store list. In total, 50,200 transaction lines, $8,777,145
in revenue, 53,707 units and 80 products across 9 categories. Prices range from $19 to $999 and the
stores trade from 10:00 to 20:00, seven days a week. The following questions are raised:

- Which branch performs best, and is the difference explained by what customers buy or by how many
  customers there are?
- On which days and at which hours do the stores actually trade?
- Which products and categories generate revenue, and how concentrated is that revenue?
- Which products are gaining ground and which are losing it compared with last year?
- Is the business growing, and does it show any seasonality?

As part of the process, the eighteen monthly files were combined with the folder connector so that new
months can be added by refreshing rather than rebuilding. Two problems in the source were corrected on
load: the transaction time column is typed as a time value in the 2025 files and as text in the 2026
files, which produces around 16,800 errors if type detection is left to Power Query; and the fact and
dimension tables use different names for the store key. A star schema was then built over a
50,200-row fact table with product, store, date and image dimensions, auto date/time was disabled and
a single date table marked, which keeps the model at 2.5MB.

Around 50 DAX measures were written. Four are worth describing. Product segmentation thresholds use
the median revenue and median units within each category rather than one global average, because
across a $19 to $999 price range a single threshold sorts products by price instead of by performance.
Rank movement compares current-period rank against prior-period rank, which surfaces products losing
position while still sitting in the top half of an absolute ranking. Month-on-month growth is
compounded rather than averaged, since an arithmetic mean of +20% and −20% returns 0% where the real
figure is −4%. Year-over-year measures are wrapped so that a 2025 filter returns "n/a", because the
calendar covers only 18 months and an unguarded measure would show a blank card or a 0% that reads as
flat performance.

The dataset contained product names but no images, so a Python script was written to generate one
photograph per product using the FLUX model through the Pollinations API. Each product row carries its
own prompt and the script appends a fixed style block to every request specifying background,
lighting, camera angle and composition, which keeps 80 images across 9 categories and 33 product types
visually consistent. Each request is seeded with the product id so the set regenerates identically.
The script skips files that already exist, retries on failure, waits for cloud storage to sync and
writes the resulting public URLs back into the product workbook. In Power BI that column is set to the
Image URL data category and shown in a report-page tooltip alongside each product's monthly trend.

The report is laid out over three pages: an executive overview with KPI cards and prior-year deltas, a
product page with a segmentation quadrant, a Pareto view and rank movement tables, and a store and
demand page with per-store scorecards and a day-by-hour revenue matrix. Slicers are synced across all
pages so filter selections survive navigation, and a custom theme was applied so that new visuals
inherit the formatting.

Some conclusions:

- The revenue gap between branches comes from traffic, not from merchandising. Long Island City takes
  39.8% of revenue, Paramus 32.1% and Red Hook 28.0%, but average order value is $175.07, $174.36 and
  $175.08 respectively, category mix differs by no more than 1.3 percentage points, and the hourly
  profiles are nearly identical.
- Weekend trading dominates. Saturday and Sunday produce 46.3% of revenue, which per calendar day is
  $26,066 against $12,079 on a weekday. Peak hour is 17:00 at all three branches on all seven days.
- Revenue is concentrated in a fifth of the range. Nineteen of the 80 products account for half of all
  revenue and the top ten for 31.9%. Sofas & Armchairs alone is 27.1% of the business at $2.38M, while
  the twenty lowest-performing products together contribute 8.3%.
- Growth is modest but positive. The first half of 2026 closed at $2,939,238 against $2,746,491 in the
  first half of 2025, up 7.0%. Monthly revenue ranges from $390,918 in February 2025 to $575,150 in
  November 2025.
- Baskets are small and consistent. Units per transaction average 1.07 and only 7.0% of baskets
  contain more than one item, which is expected for furniture but means revenue moves with footfall
  rather than with basket size.

The source contains no cost field, so no margin or profitability measure could be built, and no
list-price reference or promotion flag, so discounting could not be measured either. There is also no
customer identifier, which rules out any analysis of repeat purchase or customer segmentation. It is
proposed that cost, promotion and customer data be collected, since margin and repeat-purchase
behaviour are the two areas where this dataset currently cannot support a decision.

**Tools:** Power BI Desktop, Power Query (M), DAX, Python (requests, Pillow, openpyxl), FLUX via the
Pollinations API.

---

## Suggested tags / skills

Power BI · DAX · Power Query · Data Modeling · Data Visualization · Retail Analytics · Star Schema ·
Time Intelligence · Python · Dashboard Design

---

## Verified figures (do not publish anything not on this list)

| Metric | Value |
|---|---|
| Revenue | $8,777,145 |
| Transactions / units | 50,200 / 53,707 |
| Average order value | $174.84 |
| Period | 1 Jan 2025 – 30 Jun 2026 (546 trading days, open 7 days/week) |
| H1 2026 vs H1 2025 | $2,939,238 vs $2,746,491 → +7.0% |
| Monthly revenue range | $390,918 (Feb 2025) – $575,150 (Nov 2025) |
| Store revenue share | Long Island City 39.8% · Paramus 32.1% · Red Hook 28.0% |
| Store AOV | $175.07 · $174.36 · $175.08 |
| Category mix spread across stores | ≤ 1.3 percentage points |
| Weekend share of revenue | 46.3% |
| Revenue per weekend day vs weekday | $26,066 vs $12,079 (2.16x) |
| Trading hours / peak hour | 10:00–20:00 / 17:00 |
| Top 10 of 80 products | 31.9% of revenue |
| Products to reach 50% / 80% of revenue | 19 / 44 |
| Bottom 20 products | 8.3% of revenue |
| Largest category | Sofas & Armchairs, $2.38M (27.1%) |
| Units per transaction | 1.07 (7.0% of baskets multi-unit) |
| Generated images | 80 PNG, 768×768, FLUX via Pollinations |
