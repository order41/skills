---
name: gdrive-csv-image-matcher
description: >
  Accesses a Google Drive folder, reads all image filenames, matches them against names in a CSV file, and writes image URLs back into the matching CSV rows. Always use this skill when the user wants to match Google Drive images to a spreadsheet or CSV by name, add Drive image URLs to a CSV, link images to product rows, or populate a CSV with image URLs from a Drive folder. Trigger for phrases like "match images from Drive to my CSV", "add image URLs to my spreadsheet", "link Drive photos to my product list", "fill in image URLs from Google Drive", "match folder images to CSV rows", or any request combining a Google Drive folder, image files, and a CSV or spreadsheet.
compatibility: "Requires Google Drive MCP connection, bash_tool, and pandas Python package."
---

# Google Drive CSV Image Matcher

Reads all image filenames from a Google Drive folder, matches them by name against rows in a CSV file, and writes a Google Drive direct-access URL into the matching CSV rows.

---

## Step 0 — Understand the inputs

Before doing anything, confirm you have or can get:

1. **The Google Drive folder** — name, link, or folder ID containing the images
2. **The CSV file** — uploaded by the user OR a path/Drive link to it
3. **The name column** in the CSV — which column holds the names to match against image filenames

If any of these are unclear, ask the user before proceeding. Example:

> "Got it! To match images to your CSV I need three things:
> 1. The Google Drive folder name (or link) that contains the images
> 2. Your CSV file — please upload it or share a link
> 3. Which column in the CSV has the names I should match against? (e.g., 'product_name', 'sku', 'title')"

---

## Step 1 — Verify Google Drive connection

Use `tool_search` to confirm Google Drive MCP tools are available:

```
tool_search(query="google drive list files search")
```

If no tools are returned, tell the user:

> "Google Drive isn't connected. Please go to **Settings → Integrations** in Claude.ai and connect your Google Drive, then try again."

---

## Step 2 — Find and confirm the Drive folder

**If the user gave a folder name**, search for it:

```
google_drive_search(
  api_query="name = 'FOLDER_NAME' and mimeType = 'application/vnd.google-apps.folder'",
  semantic_query="folder named FOLDER_NAME"
)
```

**If multiple folders match**, list them and ask the user to pick one.

Note the confirmed folder's **ID** and **name**.

---

## Step 3 — List all images in the folder

Search for image files in the folder:

```
google_drive_search(
  api_query="'FOLDER_ID' in parents and (
    mimeType = 'image/jpeg' or
    mimeType = 'image/png' or
    mimeType = 'image/webp' or
    mimeType = 'image/gif' or
    mimeType = 'image/bmp' or
    mimeType = 'image/tiff' or
    mimeType = 'image/heic' or
    mimeType = 'image/avif'
  )",
  semantic_query="images photos in folder"
)
```

For each image, collect:
- `file_id`
- `file_name` (full name with extension)
- `name_stem` — filename **without extension** (used for matching)

Build a lookup dict in memory — use a **list** of file IDs per key to support multiple images sharing the same name:

```python
from collections import defaultdict
import os

# e.g. {"sunset-photo": ["FILE_ID_123", "FILE_ID_789"], "product-hero": ["FILE_ID_456"]}
image_map = defaultdict(list)
image_map_fuzzy = defaultdict(list)

for f in drive_images:
    stem = os.path.splitext(f["name"])[0]
    key = stem.strip().lower()
    key_fuzzy = key.replace("-", "").replace("_", "").replace(" ", "")
    image_map[key].append(f["id"])
    image_map_fuzzy[key_fuzzy].append(f["id"])
```

Tell the user how many images were found:

> "Found **N images** in '[Folder Name]'."

---

## Step 4 — Load the CSV

If the user uploaded the CSV, read it from `/mnt/user-data/uploads/`:

```python
import pandas as pd
df = pd.read_csv("/mnt/user-data/uploads/FILENAME.csv")
print(df.columns.tolist())
print(df.head())
```

If the user provided a Google Drive link or file ID, fetch it:

```
google_drive_search(api_query="name = 'FILENAME.csv'", semantic_query="csv file")
```
Then use `Google Drive:read_file_content` or `Google Drive:download_file_content` to get the data.

Confirm the column name that holds the names to match against (e.g., `product_name`). If unsure, show the user the column headers and ask.

The output column name is always **`parent image_urls (seperated by comma)`** — do not ask the user about this.

---

## Step 5 — Match names and build URL column

### Matching strategy

Use **case-insensitive, trimmed** matching between the CSV name column and image filename stems.

Optionally, also try fuzzy matching if exact match fails — strip common separators (`-`, `_`, spaces) before comparing as a fallback.

```python
import os
import pandas as pd
from collections import defaultdict

URL_TEMPLATE = "https://lh3.googleusercontent.com/d/{file_id}"

# Output column is always fixed:
OUTPUT_COL = "parent image_urls (seperated by comma)"

def normalize(s):
    return str(s).strip().lower()

def normalize_fuzzy(s):
    return normalize(s).replace("-", "").replace("_", "").replace(" ", "")

# image_map and image_map_fuzzy are defaultdict(list) built in Step 3
# Each key maps to a LIST of file IDs (handles multiple images with the same name)

urls = []
match_log = []  # for the summary

for val in df[name_column]:
    key = normalize(str(val))
    key_fuzzy = normalize_fuzzy(str(val))

    if key in image_map and image_map[key]:
        file_ids = image_map[key]
        # Join all matching URLs with a comma separator
        cell = ",".join(URL_TEMPLATE.format(file_id=fid) for fid in file_ids)
        urls.append(cell)
        match_log.append((val, "exact", len(file_ids)))
    elif key_fuzzy in image_map_fuzzy and image_map_fuzzy[key_fuzzy]:
        file_ids = image_map_fuzzy[key_fuzzy]
        cell = ",".join(URL_TEMPLATE.format(file_id=fid) for fid in file_ids)
        urls.append(cell)
        match_log.append((val, "fuzzy", len(file_ids)))
    else:
        urls.append("")   # no match — leave blank
        match_log.append((val, "no match", 0))

df[OUTPUT_COL] = urls
```

---

## Step 6 — Save the updated CSV

Save the updated file locally and present it to the user:

```python
output_path = "/mnt/user-data/outputs/updated_with_images.csv"
df.to_csv(output_path, index=False)
```

Then call `present_files(filepaths=[output_path])`.

---

## Step 7 — Show a summary

After saving, report results clearly:

> **✅ Done — X / Y rows matched**
>
> | Status | Count |
> |--------|-------|
> | Exact match | N |
> | Fuzzy match | N |
> | No match | N |

If there are unmatched rows, list up to 10 of the unmatched names so the user can investigate:

> "These names had no matching image:
> - `widget-blue`
> - `logo final`
> - …"

Remind the user:

> ⚠️ The `lh3.googleusercontent.com` URLs require the Drive folder to be shared as **"Anyone with the link can view"** to be publicly accessible. You can set this in Google Drive's sharing settings.

---

## Edge cases

| Situation | How to handle |
|-----------|---------------|
| CSV has no header row | Ask the user which column index (0-based) holds the names |
| Output column already exists in CSV | Ask the user whether to overwrite or use a new column name |
| Multiple images match the same name | Collect **all** matching file IDs; join their URLs with `,` into a single cell in the `parent image_urls (seperated by comma)` column |
| Filename has multiple dots (e.g. `my.product.v2.jpg`) | Strip only the last extension: `os.path.splitext` handles this correctly |
| Folder has subfolders | Process only top-level images unless user asks to recurse |
| 50+ images | Warn it may take a moment; paginate Drive searches if needed |
| CSV is very large (10k+ rows) | Use pandas for efficiency; avoid row-by-row loops |
| User wants URLs in a different format | Ask: Google Drive direct link (`lh3`), export link (`drive.google.com/file/d/ID/view`), or download link (`drive.google.com/uc?id=ID`) |

---

## URL format

All URLs use the Google Drive CDN direct-access pattern:

```
https://lh3.googleusercontent.com/d/FILE_ID
```

This is the only format used. If a row matches multiple images, all URLs are joined with `,` (no space) in a single cell.