# Furniture Sales Performance — Dashboard Design Plan

**Data as of:** 2026-06-30 · **Grain:** one row = one transaction line · **Currency:** USD

---

## 1. Source inventory (verified)

| Source | Rows | Columns |
|---|---|---|
| `Transaction\*.xlsx` (18 files, 202501–202606) | 50,200 | transaction_id, transaction_date, transaction_time, store_id, product_id, unit_price, transaction_qty |
| `Product.xlsx` | 80 | product_id, product_detail, product_category, product_type |
| `Store.xlsx` | 3 | location_id, store |
| `Product_img.xlsx` | 80 | + image_filename, image_url, image_prompt |

**Baseline numbers to validate the build against:**

- Revenue = **$8,777,145** · Transactions = **50,200** · Units = **53,707** · AOV = **$174.84**
- 2025 FY = $5,837,908 · 2026 H1 = $2,939,238 (2025 H1 = $2,746,492 → **+7.0% YoY**)
- Store split: Long Island City 19,976 / Paramus 16,168 / Red Hook 14,056 transactions
- Trading hours 10:00–20:00; peak hour 17:00 (5,947 txns); Sat+Sun = 46.3% of volume
- Price band $19–$999; `transaction_qty` ∈ {1, 2} only

### Data quirks that must be handled

1. **Join key mismatch** — fact has `store_id`, dimension has `location_id`. Rename on load.
2. **Mixed type in `transaction_time`** — stored as a *time* value in the 2025 files and as *text* in the 2026 files. Power Query's auto-detect will produce errors on ~16,800 rows. Force to text, then parse.
3. **No cost / COGS anywhere.** Gross margin, profit and profitability ranking are **out of scope** — do not build visuals that imply them.
4. **Discounting is implicit.** 10 products carry a second, lower `unit_price` at exactly −15%: product_ids 1, 2, 3, 4, 12, 16, 18, 21, 23, 58 (sofas, beds, dining tables, one patio set). ~600 discounted lines, concentrated in Nov–Dec 2025. There is no discount column — it must be derived.
5. **Calendar is 18 months, not 2 years.** Any YoY measure returns blank for all of 2025. Design for that (see §4).

---

## 2. What already exists

`Furniture-Sales-Performance.vpax` shows a working model — reuse it rather than rebuilding:

- Tables: `FACT_Transaction` (50,200), `DIM_product` (80), `DIM_store` (3), `DIM_date` (546), `DIM_img` (80), `_Measures`
- 36 measures already built: `Sales`, `#Transactions`, `Unit Sold`, `AVG Transaction Value`, plus LY / LM variants, `YoY Sales Growth %`, `Sales CY Forecast`, and a full set of `… Color` / `… Icon` helper measures for conditional formatting.
- Naming convention in force: `DIM_` / `FACT_` prefixes, Title Case column names, measures parked in `_Measures`.

**Gaps this plan closes:**

- No time-of-day dimension, despite a clean 10:00–20:00 trading window.
- No promo/discount modelling — the −15% signal is invisible today.
- Auto date/time is on (`LocalDateTable_…` present) — turn it off, it bloats the model and shadows `DIM_date`.
- No list-price column, so discount depth cannot be measured.

---

## 3. Semantic model

### 3.1 Power Query

**FACT_Transaction**
1. Get Data → Folder → `\Transaction` → Combine & Transform. Keep `Source.Name` (useful for provenance, hide later).
2. `transaction_date` → Date. `transaction_time` → **Text first**, then `Time.FromText`, then extract `Hour`.
3. Rename `store_id` → `location_id`.
4. Rename to model convention: `Date`, `Time`, `Unit Price`, `Transaction Qty`.
5. Add column `Sales = [Unit Price] * [Transaction Qty]` (materialise it — cheaper than a SUMX at 50k rows and it's already in the model).
6. Add `Hour` (whole number, 10–20).
7. Verify 50,200 rows and zero errors before closing.

**DIM_product** — rename to `Product`, `Product Category`, `Product Subcategory`. Add `List Price` = the max `unit_price` observed per product (merge from the fact, or hardcode from the price list). This is what makes discount depth measurable.

**DIM_store** — 3 rows, rename `store` → `Store`.

**DIM_img** — keep `product_id` + `final_url`; set the URL column's Data Category to **Image URL**. Note the Google Drive links need the `uc?export=view&id=…` form to render in a visual.

**DIM_date** — 2025-01-01 → 2026-06-30. Existing columns are sufficient. Mark as Date Table.

**DIM_time** (new, 11 rows) — `Hour`, `Hour Label` ("10 AM"…"8 PM"), `Daypart` (Morning 10–12 / Afternoon 12–17 / Evening 17–21), `Daypart Sort`.

### 3.2 Star schema

```
                    DIM_date ──┐
                               │ 1:*
DIM_product ──1:*── FACT_Transaction ──*:1── DIM_store
     │ 1:1                     │ *:1
  DIM_img                  DIM_time
```

All single-direction, one-to-many, filtering the fact. `DIM_img` 1:1 to `DIM_product` (or just merge it into `DIM_product` — 80 rows, no reason for a separate table).

**Model hygiene:** File → Options → Data Load → uncheck **Auto date/time**. Hide every key column (`product_id`, `location_id`, `Hour`, `Source.Name`) and all `… Color` / `… Icon` helper measures.

---

## 4. Measures to add

Existing measures cover the KPI row. Add these:

**Promo / discount**
```dax
Discounted Lines   = CALCULATE([#Transactions], FILTER(FACT_Transaction, FACT_Transaction[Unit Price] < RELATED(DIM_product[List Price])))
Discount Rate %    = DIVIDE([Discounted Lines], [#Transactions])
Sales at Full Price = CALCULATE([Sales], FILTER(FACT_Transaction, FACT_Transaction[Unit Price] = RELATED(DIM_product[List Price])))
Discount Value     = SUMX(FACT_Transaction, (RELATED(DIM_product[List Price]) - FACT_Transaction[Unit Price]) * FACT_Transaction[Transaction Qty])
```

**Ranking / contribution**
```dax
Sales Rank         = RANKX(ALLSELECTED(DIM_product[Product]), [Sales], , DESC)
% of Total Sales   = DIVIDE([Sales], CALCULATE([Sales], ALLSELECTED(DIM_product)))
Running Sales %    = -- cumulative, for the Pareto line
Top 10 Sales Share = -- concentration headline for the Product page
```

**Basket / mix**
```dax
Units per Transaction = DIVIDE([Unit Sold], [#Transactions])
Multi-Unit Rate %     = DIVIDE(CALCULATE([#Transactions], FACT_Transaction[Transaction Qty] > 1), [#Transactions])
Weekend Sales %       = DIVIDE(CALCULATE([Sales], DIM_date[Day of Week] IN {6,7}), [Sales])
```

**Store benchmarking**
```dax
Sales All Stores   = CALCULATE([Sales], ALL(DIM_store))
Store Sales Index  = DIVIDE([Sales], DIVIDE([Sales All Stores], 3))   -- 1.00 = average store
```

**Guard measure for the 18-month calendar** — wrap YoY so 2025 shows a clean message rather than blank cards:
```dax
Sales vs LY Display = IF(ISBLANK([Sales LY]), "n/a — no prior year", FORMAT([Sales vs LY], "+0.0%;-0.0%"))
```

---

## 5. Dashboard pages

Three pages, 1280×720, consistent header band and a left slicer rail synced across all pages. Each page answers one question and is self-contained — a reader who only opens page 1 still leaves informed.

### Page 1 — Executive Overview
*Question: how is the business doing right now?*

| Zone | Visual |
|---|---|
| Header | Title, `Data As Of` stamp, page nav buttons |
| KPI row | 4 cards: Sales · Transactions · Units Sold · AOV — each with LY delta, arrow icon, conditional colour (measures already exist) |
| Hero, ~55% width | **Sales trend** — line + area by Month Year, 18 points, `Sales CY Forecast` extension, reference line at the 18-month average ($487.6k) |
| Right column | **Store contribution** — horizontal bar, 3 stores, sorted, labelled as % of total |
| Right column | **Category mix** — treemap, 9 categories, top 6 coloured + Other |
| Bottom strip | **Sales by day of week** — column chart. The Sat+Sun = 46.3% spike is the headline insight of the whole dataset |
| Bottom strip | **Insight callout** — three text lines driven by measures: weekend share, top-10 product concentration, current discount rate. This is what carries the two deeper pages onto the front page. |

Everything here is a total or a share — no drillthrough, no interaction beyond cross-filtering. It should be readable in ten seconds.

### Page 2 — Product Intelligence
*Question: what sells, how concentrated are we, and is discounting working?*

This is the merged product + promotion page. Split it into two vertical bands with a thin divider.

**Band A — assortment performance (upper ~60%)**

- **Top 20 products** — bar chart, paired with an adjacent table visual carrying `DIM_img` thumbnails, Product, Sales, Units, AOV. This is where the image assets earn their place.
- **Pareto** — combo chart: product bars + `Running Sales %` line, 80% reference line. Callout card for `Top 10 Sales Share`.
- **Category → Subcategory matrix** — Sales, Units, AOV, % of Total; data bars on Sales; drillable hierarchy.
- **Price-band split** — bucket into <$50, $50–149, $150–399, $400–699, $700+; Sales vs Units on a dual axis. Exposes the volume-vs-value trade-off across an 80-product range spanning $19–$999.

**Band B — promotion impact (lower ~40%)**

The differentiator — this signal is invisible to anyone reading the data casually.

- Three small cards: `Discount Rate %`, `Discount Value`, `Sales at Full Price`.
- **Discounted-line volume by month** — column chart. The Nov–Dec 2025 spike (74 and 77 lines against a ~26/month baseline) is the story.
- **The 10 promoted products** — table: List Price, Discounted Price, full-price units, discounted units, discount value, lift indicator. Filtered to product_ids 1, 2, 3, 4, 12, 16, 18, 21, 23, 58.
- Caveat text, stated plainly: **no cost data, so this measures revenue given up, not margin impact.**

Optional bookmark toggle to expand either band to full page if the layout gets tight.

**Drillthrough target (hidden page):** single-product detail — image, list price, monthly trend, store split, full-price vs discounted units.

### Page 3 — Store & Demand
*Question: where and when do we trade, and do the three stores behave differently?*

**Band A — store comparison**

- **Store scorecard** — 3-column card layout: Sales, Transactions, AOV, `Store Sales Index` (1.00 = average store) per store.
- **Category preference by store** — 100% stacked bar. Tests whether Long Island City, Paramus and Red Hook serve genuinely different customer mixes or just different volumes.

**Band B — demand timing**

- **Heatmap matrix** — Day of Week (rows) × Hour (columns, 10–20), Sales as background colour. Single most actionable visual in the report for staffing and opening hours.
- **Hourly profile** — line chart, 10:00–20:00, one line per store. Exposes whether Red Hook's evening curve differs from LIC's.
- **Daypart split** — stacked bar, Morning / Afternoon / Evening, by store.
- **Monthly seasonality** — small line or column strip, 18 months, to sit the timing view in its seasonal context.

**Drillthrough target (hidden page):** store detail — that store's own KPI row, trend, top products, hourly curve.

---

## 6. Design system

The asset folders (`icon\`, `bgi\`, `img\`) already commit to a **Japandi / warm-earthy** palette — build the report theme to match rather than fighting it.

- **Palette:** warm off-white canvas `#F7F4EF`; card surface `#FFFFFF`; primary text / dark walnut `#575A47` (matches the supplied SVG icon fill); accent muted olive; sand and taupe for secondary series. Reserve a single saturated colour for "current period" and a muted grey for "prior period" — never colour by category where a single accent will do.
- **Categorical series:** cap at 6 visible categories + "Other". Nine full-saturation colours in one chart is unreadable.
- **Typography:** one family (Segoe UI or DIN), three sizes — 28pt KPI value, 12pt visual title, 10pt body/axis.
- **Icons:** the `_24dp_575A47_` SVG set (store, shopping_bag, donut_small, empty_dashboard, file_map_stack) maps cleanly to page navigation — use the white variants for the active state.
- **Layout:** 16px gutter, cards on a consistent grid, all visual borders off, shadows subtle. Backgrounds from `bgi\` at low opacity behind the header band only — never behind chart plot areas.
- Build this once as a **custom theme JSON** so every new visual inherits it.

---

## 7. Interactivity

- **Slicer rail (left, synced):** Year, Month, Store, Category. Use the sync-slicers pane so selections carry across all three pages.
- **Page navigation:** three buttons in the header using the icon set — `empty_dashboard` (Overview), `shopping_bag` (Product Intelligence), `store` (Store & Demand) — white variant for the active state, `#575A47` for inactive.
- **Bookmarks:** a "Reset filters" button on every page; a Sales/Units toggle on Page 2; optional band-expand toggles on Pages 2 and 3.
- **Drillthrough:** product detail (from Page 2), store detail (from Page 3). Both targets are hidden pages, not part of the three-page navigation.
- **Tooltips:** custom report-page tooltip showing a product's mini trend + image on hover.
- **Edit interactions:** set KPI cards to *not* be filtered by chart clicks — they should stay the period total.

---

## 8. Build order

1. Turn off auto date/time, then load and shape all four sources in Power Query. Verify 50,200 rows, 0 errors.
2. Build `DIM_time`; add `List Price` to `DIM_product`.
3. Wire relationships, mark `DIM_date` as the date table, hide keys.
4. Confirm existing 36 measures still resolve; add the ~13 new ones from §4.
5. **Validate against §1 baselines before building any visual.** Sales must equal $8,777,145.
6. Author the theme JSON.
7. Build pages in order **3 → 2 → 1**. Store & Demand is the fastest win and validates `DIM_time`; Product Intelligence is the heaviest page and settles the promo measures; Overview is built last so its insight callout can quote conclusions the other two pages have already proven.
8. Build the two hidden drillthrough pages.
9. Add navigation, bookmarks, tooltips and edit-interactions last.

## 9. QA checklist

- [ ] Grand total Sales = $8,777,145; Transactions = 50,200; Units = 53,707
- [ ] 2026 H1 vs 2025 H1 = +7.0%
- [ ] Every YoY card on a 2025 filter shows "n/a", not a blank or a misleading 0
- [ ] Hour axis spans 10–20 only, no 00:00 rows leaking from a bad time parse
- [ ] Discounted-line count ≈ 600 across exactly 10 product_ids
- [ ] No visual anywhere implies margin, profit or cost
- [ ] Model size sane after auto date/time removal
- [ ] All three pages read correctly at 100% zoom on a 1920×1080 screen
- [ ] Slicer selections survive navigation across all three pages
- [ ] Both drillthrough pages are hidden from the nav bar and return correctly
- [ ] Pages 2 and 3 do not scroll — if a band overflows, cut a visual rather than shrinking type below 10pt
