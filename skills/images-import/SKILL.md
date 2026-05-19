---
name: gdrive-image-optimizer
description: >
  Checks Google Drive connectivity, lets the user pick a folder, converts all images in that folder to JPEG at max 2000×2000px resolution, saves them into a new copy of the folder, and returns a list of direct-access URLs using the https://lh3.googleusercontent.com/d/FILE_ID pattern. Always use this skill when the user wants to compress, convert, or resize images stored in Google Drive, export Drive images as JPEGs, create a web-friendly copy of a Drive image folder, or generate direct image URLs from Google Drive. Trigger for phrases like "optimize my Drive images", "convert Drive folder to JPEG", "resize images in Google Drive", "get image URLs from Drive", or "make a web copy of my Drive folder".
compatibility:
  tools:
    - google_drive_search (MCP)
    - google_drive_fetch (MCP)
    - bash_tool
  python_packages:
    - Pillow (PIL)
    - requests
---

# Google Drive Image Optimizer

Converts all images in a Google Drive folder to JPEG at ≤ 2000×2000 px, saves them to a new sibling folder, and returns a list of direct-access `lh3.googleusercontent.com` URLs.

---

## Step 1 — Verify Google Drive connection

Before doing anything else, confirm the user has Google Drive connected.

Use the `tool_search` tool to find the Google Drive MCP tools:

```
tool_search(query="google drive list files")
```

If no Google Drive tools are returned, stop and tell the user:

> "It looks like Google Drive isn't connected. Please go to **Settings → Integrations** in Claude.ai and connect your Google Drive account, then try again."

If the tools are available, proceed.

---

## Step 2 — Ask the user which folder to use

Ask the user to name the Google Drive folder they want to process. For example:

> "Which Google Drive folder should I optimize? Please share the folder name (or paste a link/folder ID if you have it)."

Once you have a name or ID:

**If the user gave a folder name**, search for it:
```
google_drive_search(
  api_query="name = 'FOLDER_NAME' and mimeType = 'application/vnd.google-apps.folder'",
  semantic_query="folder named FOLDER_NAME"
)
```

**If multiple folders match**, list them and ask the user to confirm which one.

**If no folder is found**, let the user know and ask them to double-check the name.

Note the confirmed folder's **ID** and **name** — you'll need both.

---

## Step 3 — List all images in the folder

Search for image files inside the confirmed folder:

```
google_drive_search(
  api_query="'FOLDER_ID' in parents and (
    mimeType = 'image/jpeg' or
    mimeType = 'image/png' or
    mimeType = 'image/webp' or
    mimeType = 'image/gif' or
    mimeType = 'image/bmp' or
    mimeType = 'image/tiff' or
    mimeType = 'image/heic'
  )",
  semantic_query="images in folder"
)
```

Collect for each image:
- `file_id`
- `file_name`
- `mimeType`

If no images are found, let the user know and stop.

Tell the user how many images were found before proceeding:

> "Found **N images** in '[Folder Name]'. I'll convert them all to JPEG at max 2000×2000 px and save them to a new folder called '[Folder Name] – Optimized'."

---

## Step 4 — Download, convert, and re-upload images

### 4a — Install Pillow if needed

```bash
pip install Pillow requests --break-system-packages -q
```

### 4b — Download each image

Use `google_drive_fetch` (or the equivalent MCP download tool) to get each file's binary content.

If the MCP tool returns base64-encoded content, decode it in Python before processing.

### 4c — Convert and resize with Python

For each image, run this logic:

```python
from PIL import Image
import io, os

MAX_SIZE = (2000, 2000)

def convert_image(input_bytes: bytes, original_name: str) -> tuple[bytes, str]:
    """Convert image to JPEG, resize to fit within 2000x2000, return bytes + new filename."""
    img = Image.open(io.BytesIO(input_bytes))

    # Convert palette/RGBA/LA modes to RGB for JPEG compatibility
    if img.mode in ("RGBA", "LA", "P"):
        background = Image.new("RGB", img.size, (255, 255, 255))
        if img.mode == "P":
            img = img.convert("RGBA")
        background.paste(img, mask=img.split()[-1] if img.mode in ("RGBA", "LA") else None)
        img = background
    elif img.mode != "RGB":
        img = img.convert("RGB")

    # Resize only if larger than the max
    img.thumbnail(MAX_SIZE, Image.LANCZOS)

    out = io.BytesIO()
    img.save(out, format="JPEG", quality=88, optimize=True)
    out.seek(0)

    # Replace extension with .jpg
    stem = os.path.splitext(original_name)[0]
    new_name = stem + ".jpg"
    return out.read(), new_name
```

### 4d — Create the destination folder

Using the Google Drive MCP tool, create a new folder in the **same parent** as the source folder:

- Name: `[Original Folder Name] – Optimized`

Note the new folder's **ID**.

### 4e — Upload each converted image

Upload each converted JPEG to the new folder using the Google Drive MCP create/upload tool.

After uploading, note each file's **new file ID** returned by the API.

---

## Step 5 — Build and return the URL list

For each uploaded file, construct the direct-access URL:

```
https://lh3.googleusercontent.com/d/FILE_ID
```

> **Note:** These URLs work for files that are shared as "Anyone with the link can view." If the files are private, the URLs will require the viewer to be logged into a Google account that has access. Remind the user of this.

Present the results clearly:

---

### ✅ Done — [N] images optimized

Saved to Google Drive folder: **[Folder Name] – Optimized**

| # | Original filename | New filename | URL |
|---|---|---|---|
| 1 | photo1.png | photo1.jpg | https://lh3.googleusercontent.com/d/ABC123 |
| 2 | banner.webp | banner.jpg | https://lh3.googleusercontent.com/d/DEF456 |
| … | … | … | … |

---

Also offer to copy just the URLs as a plain list if the user needs them for another tool.

---

## Edge cases

| Situation | How to handle |
|---|---|
| Image is already a JPEG under 2000×2000 px | Still re-upload to the new folder; skip re-encoding to avoid quality loss — copy as-is |
| File with `.jpg` extension is actually another format | Pillow detects the real format on open; proceed normally |
| Animated GIF | Convert only the first frame to JPEG; warn the user animation will be lost |
| HEIC files | Pillow may not support HEIC without `pillow-heif`; install it with `pip install pillow-heif --break-system-packages` and register the opener: `from pillow_heif import register_heif_opener; register_heif_opener()` |
| Download fails for a file | Skip it, log the filename, and note it in the final summary |
| Folder has subfolders | Process only the top-level folder unless the user explicitly asks to recurse |
| Very large number of images (50+) | Warn the user it may take a while; process in batches of 10 and show progress |

---

## Important notes on URL accessibility

The `lh3.googleusercontent.com/d/FILE_ID` URL pattern serves files directly from Google's CDN. For the URLs to work publicly:

1. The file (or its parent folder) must be shared with **"Anyone with the link"** in Google Drive.
2. Without that permission, the URL will redirect to a Google login or return a 403 error.

If the user wants the URLs to be publicly accessible, remind them to set the folder sharing to **"Anyone with the link can view"** in Google Drive after the upload completes — this is a sharing permission change that the user must do themselves.