---
name: sell-through-analyst
description: >
  Full sell-through analysis for retailers buying from wholesale distributors, brands, sales agents.
  Use this skill whenever the user uploads one or more purchase order files (.xlsx in Order41 format)
  along with their Shopify sales data (orders CSV exports) and wants to understand how well their
  bought-in stock is performing. Triggers on phrases like "check my sell-through", "how did these
  brands perform", "which POs made money", "analyse my purchase orders", "show me sell-through by brand",
  "create a buy report", or any combination of uploading PO files + order CSVs.
  Also triggers when the user wants LinkedIn-ready cards or an anonymized version of the report.
  Always use this skill when purchase orders and sales data are both present, even if the user
  just says "check these files" or "how did it go this season".
---

# Sell-Through Analyst

Turn purchase order files and Shopify order exports into a full sell-through report: per-SKU sales,
refunds, discounts, brand P&L, a product-type heat map, a live dashboard widget, and optional
LinkedIn PNG cards with anonymised brand names.

## What you need from the user

| File | Format | Notes |
|------|--------|-------|
| 1+ purchase order files | `.xlsx` (Order41 format) | Header row at row 12 (0-indexed 11), data from row 13 |
| Sales order exports | Shopify CSV (`orders1.csv` etc.) | Can be split across multiple files |
| `products.csv` | Shopify product export | For Shopify status (active / draft / not listed) |

If the user only uploads PO files without order CSVs, tell them you need the orders data too and
ask them to upload it. Do not guess or fabricate sales figures.

## Step 1 — Parse the purchase orders

Use `scripts/parse_pos.py` to extract all POs into a unified dataframe.

```bash
python scripts/parse_pos.py <po_file1.xlsx> [po_file2.xlsx ...] --out /tmp/pos.csv
```

The script handles the Order41 Excel layout automatically. It reads:
- Col 5 (Product ID) → SKU
- Col 6 (Name) → product name
- Col 8 (Size) → size
- Col 18 (Qty) → units ordered
- Col 19 (Price) → cost price
- Col 20 (MSRP) → retail price
- Col 22 (Type) → product category
- Col 23 (Supplier) → brand (may be blank for some distributors — infer from rider/product names if needed)
- Row 7 col 1/2 → PO total (verification)
- Row 0 col 3 → PO date

**Brand inference**: if the Supplier column is blank, look at the product names and check whether
the orders data confirms the brand (e.g. "de Keyzer" → Quasi Skateboards from Shopify Vendor field).

## Step 2 — Match against sales orders

Use `scripts/match_orders.py` to find every sale for each PO SKU.

```bash
python scripts/match_orders.py /tmp/pos.csv <orders1.csv> [orders2.csv ...] \
  [--products products.csv] --out /tmp/matched.csv
```

The script:
- Loads all orders CSVs and concatenates them
- Joins on `Lineitem sku` = PO SKU
- Treats `Financial Status` of `refunded` / `voided` as returns; everything else (including NaN rows,
  which are additional line items on a paid order) as sold
- Captures `Lineitem discount` per line
- Looks up `Status` from products.csv for each SKU (active / draft / not in store)
- Outputs one row per PO SKU with: net_sold, refunded_qty, net_rev, disc, shopify_status

## Step 3 — Decide which POs to include

After matching, print a summary table so the user can see which POs have sales and which have zero.
Ask: "These POs had no sales — include them anyway, or skip?" unless the user already told you.

Common reasons for zero sales:
- Products not yet listed in Shopify (check shopify_status = "not in store" or "draft")
- PO is for a future season and goods haven't arrived yet
- SKU format mismatch (try a name-based fallback search before concluding zero)

For the final report, include all POs the user confirms, flagging zero-sales ones prominently.

## Step 4 — Build the Excel report

Use `scripts/build_excel.py` to generate the workbook.

```bash
python scripts/build_excel.py /tmp/matched.csv --out /tmp/report.xlsx \
  [--title "My Season Report"] [--date "2026-06-10"]
```

The workbook has these sheets in order:

1. **Brand Overview** — one row per PO: units ordered/sold/refunded/remaining, PO cost, net revenue,
   net profit (green if positive, red if negative), sell-through % (colour-coded), discount total,
   Shopify status. Totals row at bottom. Flag zero-sales POs in amber.

2. **Heat Map** — sell-through % by product type (rows) × brand (columns). Colour scale:
   green ≥100%, yellow 50–99%, orange <50%, red 0% (listed), amber 0% with ⚠ (not in store).

3. **One tab per brand** — full SKU-level detail: cost, MSRP, markup, qty, net sold, refunded,
   sell-through %, net revenue, discount, Shopify status. Cells with discounts highlighted amber.
   Cells with "not in store" highlighted red. Draft highlighted amber.

## Step 5 — Render the dashboard widget

Use the `show_widget` tool to render an inline dashboard. Include:

- Amber warning banner for any zero-sales, in-season POs
- 8 KPI cards: total invested, net revenue, net P&L, best brand profit, units ordered, units sold, discounts, remaining @ MSRP
- Sell-through bar per brand (bars capped at 100% visually; label shows real %; "not listed" shown separately)
- Revenue vs cost grouped bar chart (Chart.js)
- Units ordered vs sold grouped bar chart (Chart.js)
- Heat map table (colour-coded cells)
- Two action columns: "Do now" and "Buying guidance"

Brand colours: assign a distinct dark colour per brand and keep it consistent across all charts/tables.

## Step 6 — LinkedIn PNG cards (optional)

If the user asks for LinkedIn cards or mentions publishing/sharing results, ask whether they want
brands anonymised (replace with Brand A, Brand B, …). Then run:

```bash
python scripts/make_cards.py /tmp/matched.csv --out /tmp/ [--anonymize]
```

This produces four 1080×1080 PNG cards:
1. **KPI overview** — headline metrics + sell-through bars per brand
2. **Brand comparison** — cost vs revenue bars + units table
3. **Heat map** — product type × brand sell-through
4. **Key learnings** — per-brand bullet points + process insight

Present all four with `present_files`.

## Key calculations

```
net_sold     = units sold (paid + NaN status) — does NOT subtract refunds
refunded_qty = units with status refunded / voided
sell_through = net_sold / qty_ordered  (can exceed 1.0 if pre-existing stock covered extra sales)
net_rev      = sum of (qty × price − line_discount) for sold rows
gross_margin = net_rev − (min(net_sold, qty_ordered) × cost)
net_profit   = net_rev − po_cost
```

NaN financial status rows are valid sales — they are additional line items on a multi-line order
where only the first line carries the order-level status.

## Handling edge cases

- **Oversold SKUs** (net_sold > qty_ordered): flag in the detail tab but do not cap to 100%;
  this signals pre-existing stock that absorbed extra demand. Still a positive signal.
- **Partially refunded orders**: count the full qty as sold (item not fully returned).
- **Missing brand/supplier field**: infer from product names + Shopify Vendor lookup.
- **Multiple FA POs**: label them by season (FA SU25, FA FA25) not just brand name.
- **Deck sales online vs in-store**: if decks show 0% online across multiple POs, note this
  as a likely channel split (in-store buyers, not online).

## Communication style

After presenting the dashboard and Excel file, give a short plain-English summary:
- Which brand is the standout and why
- Which brands are still in the red and the most likely reason
- The single most urgent action item
- One forward-looking buying recommendation

Keep it to 4–5 sentences. The visuals carry the detail; the text just orients the user.