# Furniture Sales Performance — post draft

Audience: Power BI instructor / course community. Warm, friendly, asking for feedback.

---

## Post

> Hi everyone!
>
> I just finished my first Power BI dashboard, and I'd genuinely love some feedback on it 🙏
>
> Honestly, this isn't my first time opening Power BI — but the end-to-end process always felt blurry
> to me. The tips and design philosophy from sis @Anh Leimer made it finally click, and saved me so
> much time. It really was a shortcut.
>
> The project: an 18-month furniture retail dataset — 50,200 transactions across 3 stores and 80
> products — built into three pages: Executive Overview, Product Intelligence, and Store & Demand.
>
> **A few techniques I tried out:**
>
> - The dataset came with no product images, so I generated all 80 with AI from a Python script — the
>   same style block on every prompt so they look like one catalog, and seeded by product_id so the
>   whole set is reproducible
> - A product segmentation quadrant that splits on the median *within each category* rather than the
>   overall average — with prices from $19 to $999, one global average threshold just sorts by price
> - Rank-movement measures, to surface which products are climbing and which are quietly slipping
> - A custom tooltip page that shows the product photo next to its monthly trend
>
> **One limitation I ran into:**
>
> The dataset has no cost or discount data, so there's no honest way to build a margin or
> profitability view — and for a sales dashboard, that's usually the first thing a stakeholder asks
> for. I chose to leave the gap rather than estimate it. If a cost column were ever added, I think it
> would open the door to a lot more variety in what people can build from the same brief 🙂
>
> Would love to hear what you'd do differently — modelling, DAX, layout, anything at all. Thank you!
>
> #PowerBI #DAX #DataAnalytics #LearningInPublic

---

## Optional extras

Swap in or add if you want a little more pull:

- **One insight as a hook:** "The nicest surprise: the three stores turned out to be the same shop at
  three sizes — average order value is $175 in all three, within 72 cents of each other."
- **A second insight:** "Saturday and Sunday bring in 46.3% of revenue, and 17:00 is the peak hour in
  every store, every day."
- **A data-cleaning war story:** "One column was stored as a time value in the 2025 files and as text
  in the 2026 files — about 16,800 error rows if you let Power Query auto-detect it."
- **A specific question to invite replies:** "One thing I'm unsure about — I have about 25 helper
  measures that exist only to drive colours and icons. Is that normal, or should calculation groups
  be doing that job?"

### Note on the AI-image credit

Your draft says "ChatGPT". The script in this folder generates the images through the Pollinations
API using the **FLUX** model — ChatGPT isn't in that loop unless you used it to write the 80 prompts.
The post above just says "with AI", which is accurate either way and reads fine. If you'd rather name
the tool, say **FLUX** for the images (and "prompts drafted with ChatGPT" if that's what happened) —
an instructor may well ask which model you used.

---

## Full technique inventory (reference)

### Data acquisition and enrichment

1. **AI-generated product catalog.** 80 images, 768×768 PNG, 22 MB total, generated with FLUX through
   the Pollinations API by a purpose-built Python script.
2. **Two-layer prompting for consistency.** Each product row carries its own prompt; the script
   appends one shared REFERENCE_STYLE constant to every request (seamless white background, high-key
   studio lighting, three-quarter front angle, no props, no text). That constant is what makes 80
   images across 9 categories and 33 product types read as a single catalog.
3. **Deterministic generation via seed = product_id.** Same product, same image, every run.
4. **Idempotent, resumable pipeline.** Skips existing files, --start-at / --count batching, 3 retries
   with escalating backoff, re-encoded through Pillow to optimised RGB PNG.
5. **Automated publish-and-link-back.** Polls Google Drive until all 80 files sync, scrapes the folder
   listing to map each filename to its public URL, writes those URLs back into the workbook.
6. **Folder connector over 18 monthly Excel files.**

### Power Query

7. **Mixed-type transaction_time fix** — time values in 2025 files, text in 2026 files; forced to text
   and parsed explicitly (auto-detect gives roughly 16,800 error rows).
8. **Join-key mismatch resolved on load** (store_id to location_id).
9. **Sales materialised as a column** rather than a query-time SUMX.
10. **List Price derived per product** as the max observed unit price.

### Model

11. Star schema — FACT_Transaction (50,200) with DIM_product, DIM_store, DIM_date, DIM_img, _Measures.
12. Auto date/time disabled, single marked date table, keys hidden.
13. DIM_img joined 1:1 to DIM_product, image_url set to the **Image URL** data category.
14. DIM_ / FACT_ prefixes and Title Case columns throughout.

### DAX

15. **Median-within-category segmentation** — Median Sales per ProductCat and Median Units per
    ProductCat as quadrant thresholds; Segment, Segment Detail, Segment Revenue Share, Segment Product
    Share, Segment YoY Growth.
16. **Rank movement** — Rank Current, Rank PP, Rank Movement Label, Rank Movement Color.
17. **Pareto running share** on a line-and-column combo with an 80% reference line.
18. **Avg MoM Growth % (Compound)** rather than an arithmetic mean of growth rates.
19. **Sales CY Forecast** extending the trend line.
20. **Dynamic hero measures** — Hero Product, Hero Sales, Hero Image.
21. **Around 25 colour and icon measures** so conditional formatting is defined once in the model.
22. **YoY guard** for the 18-month calendar — 2025 reads "n/a", never blank or 0%.
23. **Data As Of** stamp on every page.

### Report

24. Three pages at 1920×1080 — Executive Overview, Product Intelligence, Store & Demand.
25. **Custom report-page tooltip** (250×430) pairing the product image with its MoM trend.
26. Slicers synced across all pages, page navigator, four bookmarks.
27. **Custom theme** in a warm Japandi palette (#F7F4EF canvas, #575A47 text and icons), one type
    family at three sizes, matched SVG icon set with active and inactive states.
28. **Deliberate omission:** no cost or COGS in the source, so no margin, profit or profitability
    visual anywhere.

---

## Verified figures (safe to quote)

| Metric | Value |
|---|---|
| Revenue | $8,777,145 |
| Transactions / units | 50,200 / 53,707 |
| Average order value | $174.84 |
| Period | 1 Jan 2025 – 30 Jun 2026 (546 trading days, open 7 days/week) |
| H1 2026 vs H1 2025 | $2,939,238 vs $2,746,491 → +7.0% |
| Store revenue share | Long Island City 39.8% · Paramus 32.1% · Red Hook 28.0% |
| Store AOV | $175.07 · $174.36 · $175.08 (spread $0.72) |
| Category mix spread across stores | ≤ 1.3 percentage points |
| Weekend share of revenue | 46.3% |
| Revenue per weekend day vs weekday | $26,066 vs $12,079 → 2.16x |
| Trading hours / peak hour | 10:00–20:00 / 17:00 |
| Top 10 of 80 products | 31.9% of revenue |
| Products to reach 50% / 80% of revenue | 19 / 44 |
| Bottom 20 products | 8.3% of revenue |
| Largest category | Sofas & Armchairs, $2.38M (27.1%) |
| Generated images | 80 PNG, 768×768, FLUX via Pollinations |

---

## Don't claim

- Anything about **margin, profit or discounting** — no cost column, no discount visual in the report.
- **"Real-time" or "live"** — this refreshes over monthly Excel extracts.
- **Customer-level behaviour** — no customer ID in the data.
- Don't present the AI images as **real product photography**. Say they are generated.
