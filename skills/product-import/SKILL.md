---
name: ultimate-importer
description: >
  Reads product data from CSV, XLSX, PDF, or image files and creates a standardized Order41 import CSV. Always use this skill when a user uploads any file and mentions importing, products, or Order41 — even if they just say 'turn this into a CSV' or 'prep this for Order41'. Also trigger for phrases like 'wholesale file', 'product list', or any request to reformat product data for upload.
---

# Product Import File Creator

You are helping the user create an Order41-compatible import CSV file from uploaded product data.

---

## Step-by-step process

1. **Read the source file** — use the appropriate method for the file type:
   - CSV/XLSX: read directly with bash/python
   - PDF: extract text with `pdfplumber` or `pdfminer`
   - Image: describe what you see and extract data manually
2. **Identify products and variants** — group rows by parent product. Each size/color combination = one variant row.
3. **Map fields** using the field reference below
4. **Write the CSV** to `/mnt/user-data/outputs/import.csv`
5. **Present the file** to the user with a short summary (X products, Y variants, any fields left blank)

---

## Output CSV structure

One row per variant. The header must be exactly:

parent_urlkey,variant_id,variant_row_order,name,info,type,parent_sku,sku,inventory_stock,cost,cost_currency,wholesale_price,MSRP,currency,Size,Color,Delivery,GTIN,Material,Fiber,Origin,Taric,Delivery_date,parent_image_urls,image_urls

---

## Field reference

| Column | Description | Required | Default if missing |
|---|---|---|---|
| `parent_urlkey` | URL-safe slug for the parent product, e.g. `blue-linen-shirt` | Yes | Slugify the product name |
| `variant_id` | Assigned by Order41 — always leave blank | — | _(empty)_ |
| `variant_row_order` | Row order within the parent group, starting at 0 | Yes | Increment from 0 |
| `name` | Product name | Yes | — |
| `info` | Product description | No | _(empty)_ |
| `type` | Product category/type e.g. `tshirt`, `jacket`, `pants` | No | _(empty)_ |
| `parent_sku` | SKU shared across all variants of the same product | Yes | — |
| `sku` | Unique SKU for this variant | Yes | — |
| `inventory_stock` | Stock count | No | _(empty)_ |
| `cost` | Wholesale cost (number only) | No | _(empty)_ |
| `cost_currency` | Currency code e.g. `EUR`, `USD` | No | _(empty)_ |
| `wholesale_price` | Price shown to wholesale buyers | No | _(empty)_ |
| `MSRP` | Retail price / RRP | No | _(empty)_ |
| `currency` | Currency for wholesale_price and MSRP | No | Same as cost_currency |
| `Size` | Size value e.g. `S`, `M`, `42`, `One Size` | No | _(empty)_ |
| `Color` | Color name e.g. `White`, `Navy` | No | _(empty)_ |
| `Delivery` | Delivery window e.g. `Q1 2025` | No | _(empty)_ |
| `GTIN` | Barcode (EAN/UPC) | No | _(empty)_ |
| `Material` | Primary material e.g. `100% Cotton` | No | _(empty)_ |
| `Fiber` | Secondary fiber content if separate from material | No | _(empty)_ |
| `Origin` | Country of origin e.g. `Portugal` | No | _(empty)_ |
| `Taric` | Customs/HS tariff code | No | _(empty)_ |
| `Delivery_date` | Specific delivery date if known | No | _(empty)_ |
| `parent_image_urls` | Comma-separated image URLs for the parent product | No | _(empty)_ |
| `image_urls` | Comma-separated image URLs for this specific variant | No | _(empty)_ |

---

## Grouping products into parent/variant rows

Products with the same `parent_sku` belong to the same parent. They share: `parent_urlkey`, `name`, `info`, `type`, `parent_sku`, `parent_image_urls`.

They differ per variant: `sku`, `Size`, `Color`, `inventory_stock`, `GTIN`, `image_urls`.

`variant_row_order` starts at `0` for the first variant of each parent and increments by 1.

---

## Edge cases

- **Single-SKU product (no variants):** Write one row. Leave `Size` and `Color` blank.
- **Missing price/currency:** Leave both `cost` and `cost_currency` blank — don't guess.
- **Multiple delivery dates:** Use the earliest date in `Delivery`, put the rest in `Delivery_date` or a note.
- **Images not available:** Leave image columns blank — do not fabricate URLs.
- **Ambiguous parent grouping:** If you cannot confidently group variants, ask the user before writing the file.

---

## Example output

parent_urlkey,variant_id,variant_row_order,name,info,type,parent_sku,sku,inventory_stock,cost,cost_currency,wholesale_price,MSRP,currency,Size,Color,Delivery,GTIN,Material,Fiber,Origin,Taric,Delivery_date,parent_image_urls,image_urls
blue-linen-shirt,,0,Blue Linen Shirt,Relaxed fit summer shirt,shirt,BLS-001,BLS-001-S-BLU,12,45,EUR,90,180,EUR,S,Blue,Q2 2025,,100% Linen,,Portugal,,,,
blue-linen-shirt,,1,Blue Linen Shirt,Relaxed fit summer shirt,shirt,BLS-001,BLS-001-M-BLU,8,45,EUR,90,180,EUR,M,Blue,Q2 2025,,100% Linen,,Portugal,,,,
wool-trousers,,0,Wool Trousers,,trousers,WT-220,WT-220-38,5,,,120,240,EUR,38,,,,,80% Wool 20% Polyester,,Italy,,,,
