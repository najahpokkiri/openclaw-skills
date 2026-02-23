---
name: planet-order
description: "Order satellite imagery from Planet Explorer via browser automation. User sends AOI file (GeoJSON/KML/Shapefile) + date range via Telegram. Skill downloads AOI, uploads it to Planet Explorer, selects minimum scenes covering the AOI, orders full scene delivery, and sends PNG via Telegram. Trigger: user sends a file attachment with dates, or mentions a saved location + date range."
metadata: {"openclaw":{"requires":{"bins":["gdal_translate","gdalwarp","gdal_merge.py","ogr2ogr","curl"]},"emoji":"🛰️"}}
---

# Planet Satellite Imagery Order Skill

Receives AOI from Telegram → uploads to Planet Explorer → selects right scenes → **visually checks cloud cover over target facility** → orders full scenes → polls until ready → downloads → mosaics + clips locally → converts to PNG → sends via Telegram.

Credentials in env: PL_EMAIL, PL_PASSWORD, TELEGRAM_BOT_TOKEN.

---

## Output Rules

- Work silently while doing setup and scene search
- **Allowed messages:**
  1. Pre-order notice (before placing order): scene cloud % + estimated facility cloud %, date, scene count
  2. Every 5 minutes during polling: one brief status update (see Step 7)
  3. Final: send the PNG file with caption
- If no clear date found: message user with best facility cloud estimate and ask to confirm before ordering
- Do NOT send screenshots to the user

---

## Key Principle: Clip to Facility, Not the Whole AOI

The AOI file may include surrounding land, trees, or buffer zones the user does not care about. The target is the **facility or structure** inside the AOI — an airport, building complex, industrial site, etc. All cloud checks and all clips must focus on that target, ignoring irrelevant surrounding area.

---

## Triggering — PRIORITY SKILL

**ALWAYS activate this skill when:**
- User sends ANY .geojson, .kml, or .zip file — always assume they want imagery ordered
- User says "order", "get image", "satellite", "Planet" + any location or date
- User says "crop last image" — crop-only mode
- User says "check my Planet orders" or "download completed orders"

**If a GeoJSON is received with no date range:** ask ONLY for the date range, then proceed immediately.
**If a GeoJSON is received with a date range:** start immediately, no questions.

---

## Saved Locations

| Name | SW (lon, lat) | NE (lon, lat) |
|------|--------------|--------------|
| Isa Town | 50.54, 25.92 | 50.56, 25.94 |
| Isa Bahrain | 50.5, 26.1 | 50.65, 26.25 |

---

## Step 1 — Download AOI File from Telegram

    # Get file path from Telegram
    curl -s "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/getFile?file_id=FILE_ID"
    # Returns: { "result": { "file_path": "documents/file_123.geojson" } }

    # Download file
    curl -s -o /home/openclaw/planet_orders/aoi/incoming.geojson \
      "https://api.telegram.org/file/bot${TELEGRAM_BOT_TOKEN}/documents/file_123.geojson"

Convert if needed:
    ogr2ogr -f GeoJSON aoi.geojson input.kml    # KML
    ogr2ogr -f GeoJSON aoi.geojson input.shp    # Shapefile
    # GeoJSON: use directly

---

## Step 2 — Login to Planet Explorer

- Open https://www.planet.com/explorer/ in headless browser
- Click Sign In if not logged in; enter PL_EMAIL + PL_PASSWORD from env
- Wait 10 seconds (heavy React SPA)
- Confirm logged in before continuing

---

## Step 3 — Upload AOI to Planet Explorer

1. Find the AOI upload/import button in the map toolbar (folder icon near draw tools)
2. Click it to open file input, upload the GeoJSON using the browser upload tool:
   `openclaw browser upload /home/openclaw/planet_orders/aoi/aoi.geojson`
3. Wait for AOI polygon to appear on map
4. Fallback: draw bounding box manually using the rectangle draw tool

---

## Step 4 — Apply Filters and Search

- Set date range from user message
- Cloud cover <= 20% (default)
- Imagery type: PlanetScope / PSScene, 3-band Visual RGB
- Run search and wait for results

---

## Step 5 — Select Scenes (COST CONTROL + FACILITY CLOUD CHECK)

**Goal: fewest scenes from best single day covering >= 90% of AOI, with clear sky over the target facility itself.**

### 5a — Group and rank candidate dates
1. Only scenes that actually intersect AOI geometry
2. Group by date; find minimum scenes for >= 90% coverage
3. Rank: coverage % desc → scene cloud % asc → recency

### 5b — Visual facility cloud check (MANDATORY for each candidate date)

Planet's scene-level cloud % covers the whole tile, not just your target. You MUST visually verify.

**Important:** The AOI may be a large polygon containing surrounding trees, fields, or open land. You only care about the **primary target facility** (airport, building complex, industrial site, etc.) visible within the AOI. Clouds over surrounding vegetation or non-target land do NOT count.

For each top candidate date (up to 3):
1. Click the scene in Planet Explorer to select/preview it
2. Zoom the map to the AOI bounding box so the target facility fills the viewport
3. Take a browser screenshot of the map at this zoom level
4. Visually locate the **primary target structure/facility** within the AOI
5. Estimate what percentage of the **target facility itself** is obscured by cloud or cloud shadow — ignore cloud over surrounding land, trees, or fields
6. Record: date, scene cloud %, estimated facility cloud %

**Decision rule (cloud over target facility only):**
- Facility cloud estimate < 30%: ✅ proceed with this date
- Facility cloud estimate 30–70%: ⚠️ borderline — prefer a clearer date if available
- Facility cloud estimate > 70%: ❌ skip this date, try next candidate

**If all candidates have facility cloud > 70%:**
- Pick the date with the lowest facility cloud estimate
- Message the user:
  ```
  ⚠️ Best available date: [YYYY-MM-DD]
  Scene cloud: [X]% | Estimated facility cloud: [Y]%
  No fully clear date found in range. Proceed anyway? (reply yes/no)
  ```
- Wait for user confirmation before ordering

### 5c — Pre-order notification (once clear date found)

    Found coverage for [location]
    Best date: [YYYY-MM-DD]
    Scenes to order: [N] (covers [X]% of AOI)
    Scene cloud cover: [Y]%
    Estimated facility cloud cover: [Z]%
    Ordering now...

---

## Step 6 — Place Order (Full Scenes, No Clip)

Planet charges per clip operation — clipping on Planet's side wastes money. Order **full scenes** instead and clip locally for free. Full scene TIFs are also kept permanently for future re-cropping without re-ordering.

**Order settings:**
- Select chosen scenes
- Order name format: DDMMYYYY_LocationName_Nsc  (e.g. 20022026_IsaTownTest_2sc)
- Bundle: Visual (RGB GeoTIFF)
- **Do NOT enable Clip to AOI** — order full scenes only
- Submit order
- Note the order ID and order URL from the confirmation
- Record the order start time (for elapsed time tracking in Step 7)

---

## Step 7 — Poll Until Ready and Download (DO NOT STOP UNTIL DONE)

After ordering, Planet redirects to the order detail page:
  https://insights.planet.com/data/orders/[ORDER-ID]

**Poll this page every 30 seconds. Every 5 minutes, send ONE status message to Telegram:**

    Order processing... [X] min elapsed. Planet usually takes 15–30 min.

Replace [X] with actual minutes elapsed since order was placed (round to nearest 5).
Send at: 5 min, 10 min, 15 min, 20 min, etc. — until order completes.

**Poll steps:**
1. Take a browser snapshot of https://insights.planet.com/data/orders/[ORDER-ID]
2. Look for status indicators: "success", "complete", "ready", "Download", green checkmark
3. Also check https://www.planet.com/account/#/orders for the order list status

**When order shows as complete/success:**
1. Find the Download button or download links on the order page
2. Click Download — Planet provides a zip file or individual GeoTIFF links
3. Use `openclaw browser waitfordownload` or `openclaw browser download` to save the file
4. Save full scene TIFs to the named scenes directory:

        mkdir -p ~/planet_orders/[ORDER_NAME]/scenes/
        # Move/extract downloaded TIFs here
        # e.g. unzip order.zip -d ~/planet_orders/[ORDER_NAME]/scenes/

**Alternative download via curl using browser cookies:**
If clicking download does not work, extract cookies and use curl:
    # Get cookies via openclaw browser cookies command
    # Then: curl -b "cookie_string" -L "download_url" -o output.zip

**Timeout: 20 minutes.** If exceeded, notify user with the order URL to check manually.

**NEVER go idle while waiting. Poll continuously. Do not wait for the user to check in.**

---

## Step 8 — Local Mosaic + Clip to PNG

Full scenes are stored locally. Mosaic all scenes first (fills any gaps between tiles), then clip to AOI, then convert to PNG. The output PNG is named after the order.

    ORDER_DIR=~/planet_orders/[ORDER_NAME]
    SCENES_DIR=$ORDER_DIR/scenes
    AOI=/home/openclaw/planet_orders/aoi/aoi.geojson

    # 1. Mosaic all full scenes (gdal_merge fills gaps between tiles — no black seams)
    gdal_merge.py -o $ORDER_DIR/mosaic.tif $SCENES_DIR/*.tif

    # 2. Clip mosaic to AOI (no -dstalpha — avoids black/transparent areas at edges)
    gdalwarp -cutline $AOI -crop_to_cutline \
             -dstnodata "0 0 0" \
             $ORDER_DIR/mosaic.tif $ORDER_DIR/clipped.tif

    # 3. Convert to PNG with order-specific name
    gdal_translate -of PNG -scale $ORDER_DIR/clipped.tif \
                   /home/openclaw/planet_orders/output/[ORDER_NAME].png

    # If downloaded as zip, extract to scenes dir first:
    unzip order.zip -d $SCENES_DIR/

**Why this approach:**
- `gdal_merge.py` mosaics scenes without gaps or black seams between tiles
- No `-dstalpha` means no alpha channel is added, so edge areas are black fill (not transparent)
- `-dstnodata "0 0 0"` fills any uncovered edge pixels with black instead of leaving them as nodata
- Full scenes are preserved in `scenes/` for future re-clipping to any sub-AOI

---

## Step 9 — Upload and Send via Telegram

**Output file is always `/home/openclaw/planet_orders/output/[ORDER_NAME].png`.**

**Never send any PNG file that predates the current order's start time.** Always verify the file timestamp is newer than when the order was placed before sending.

### 9a — Upload to catbox.moe (high-quality permanent link)

    CATBOX_URL=$(curl -4 -s -F "fileToUpload=@/home/openclaw/planet_orders/output/[ORDER_NAME].png" -F "reqtype=fileupload" https://catbox.moe/user/api.php)
    echo "Upload URL: $CATBOX_URL"

catbox.moe returns a direct URL like `https://files.catbox.moe/xxxxxx.png`. If upload fails, proceed to 9b without a link.

### 9b — Copy to workspace and send file via Telegram

OpenClaw only allows sending local files from the workspace directory:

    cp /home/openclaw/planet_orders/output/[ORDER_NAME].png \
       /home/openclaw/.openclaw/workspace/[ORDER_NAME].png

Then send the file:

    curl -s \
      -F "chat_id=CHAT_ID" \
      -F "document=@/home/openclaw/.openclaw/workspace/[ORDER_NAME].png" \
      -F "caption=ORDER_NAME | DATE | N scenes | CLOUD% cloud | WxH px" \
      "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendDocument"

### 9c — Send the download link as a message

    curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
      -d "chat_id=CHAT_ID" \
      -d "text=Download (full quality): $CATBOX_URL"

If PNG > 50MB: compress first then re-copy:

    convert -resize 50% /home/openclaw/planet_orders/output/[ORDER_NAME].png \
            /home/openclaw/planet_orders/output/[ORDER_NAME].png
    cp /home/openclaw/planet_orders/output/[ORDER_NAME].png \
       /home/openclaw/.openclaw/workspace/[ORDER_NAME].png

---

## Step 10 — Log the Order

Append to `/home/openclaw/planet_orders/orders.json` after every successful delivery:

    python3 -c "
    import json, os
    log = '/home/openclaw/planet_orders/orders.json'
    entries = json.load(open(log)) if os.path.exists(log) else []
    entries.append({
        'date': 'IMAGERY_DATE',
        'ordered_at': 'TIMESTAMP',
        'name': 'ORDER_NAME',
        'location': 'LOCATION_NAME',
        'file': '[ORDER_NAME].png',
        'size_mb': FILESIZE_MB,
        'catbox_url': 'CATBOX_URL',
        'sent': True
    })
    json.dump(entries, open(log, 'w'), indent=2)
    "


## Step 11 — Confirm and Go Idle

Send summary: order name, date, scene cloud %, facility cloud %, scenes, file size, dimensions.
Then go fully idle.

---

## Crop Mode (any time)

User sends "Crop last image to [coords]" or attaches a sub-AOI. Use the stored full scenes in `~/planet_orders/[ORDER_NAME]/scenes/` — no need to re-order:

    # Re-mosaic and clip to new sub-AOI:
    gdal_merge.py -o /tmp/mosaic_temp.tif ~/planet_orders/[ORDER_NAME]/scenes/*.tif
    gdalwarp -cutline sub.geojson -crop_to_cutline \
             -dstnodata "0 0 0" \
             /tmp/mosaic_temp.tif /tmp/clipped_temp.tif
    gdal_translate -of PNG -scale /tmp/clipped_temp.tif \
                   /home/openclaw/planet_orders/output/[ORDER_NAME]_crop.png

    # Bounding box crop:
    gdal_translate -projwin <ulx> <uly> <lrx> <lry> -of PNG -scale input.tif cropped.png

---

## Error Handling

- AOI upload fails: draw bounding box manually from AOI coordinates
- No scenes found: expand cloud cover to 30%, extend date range +/-14 days, retry once
- Coverage < 90%: notify user, report best %, ask to proceed
- Login fails: notify user to check credentials
- Download button not found: try navigating directly to insights.planet.com/data/orders/[ID]
- Order timeout > 20 min: notify user with order URL
- PNG > 50MB: compress and resend
