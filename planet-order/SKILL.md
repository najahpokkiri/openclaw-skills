---
name: planet-order
description: "Order satellite imagery from Planet Explorer via browser automation. User sends AOI file (GeoJSON/KML/Shapefile) + date range via Telegram. Skill downloads AOI, uploads it to Planet Explorer, selects minimum scenes covering the AOI, orders full scene delivery, and sends PNG via Telegram. Trigger: user sends a file attachment with dates, or mentions a saved location + date range."
metadata: {"openclaw":{"requires":{"bins":["gdal_translate","gdalwarp","ogr2ogr","curl"]},"emoji":"🛰️"}}
---

# Planet Satellite Imagery Order Skill

Receives AOI from Telegram → uploads to Planet Explorer → selects right scenes → **visually checks cloud cover over target facility** → orders full scenes → polls until ready → downloads → mosaics + clips locally → converts to PNG → sends via Telegram.

Credentials in env: PL_EMAIL, PL_PASSWORD, TELEGRAM_BOT_TOKEN.

---

## Output Rules

**Send one short message at each stage — one line only, no walls of text.**

**ALL stage messages MUST be sent via curl commands. Do NOT use any native OpenClaw tool to send them.**

| Stage | Message |
|-------|---------|
| AOI received | `📥 Got your AOI. Uploading to Planet Explorer...` |
| Search running | `🔍 Searching [date range]...` |
| Clear date found | Pre-order notice (date, scenes, cloud %) — see Step 5c |
| Order submitted | `🛒 Order placed ([N] scenes). Waiting for Planet to process...` |
| Every 5 min polling | `⏳ Order processing... [X] min elapsed. Planet usually takes 15–30 min.` |
| Download complete | `✂️ Download complete. Mosaicking and clipping...` |
| Converting | `🖼️ Converting to PNG...` |
| Done | Send the PNG file with caption (no "clipped to AOI") |
| After PNG sent | `Download (full quality): [CATBOX_URL]` **(MANDATORY — send as separate message)** |

- One line per message — no multi-line status walls
- If an error occurs at any stage: one-line error message, then stop
- If no clear date found: ask user to confirm before ordering (see Step 5b)
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

## Step 0 — Session Pre-Flight Gate

Before doing anything else, check whether a session is already active:

Run this shell command to get an unambiguous lock status:

```bash
python3 -c "
import json, sys, os, datetime
f = '/home/openclaw/.openclaw/workspace/current-session.json'
if not os.path.exists(f):
    print('LOCK:none')
    sys.exit(0)
try:
    d = json.load(open(f))
    if not isinstance(d, dict) or not d.get('file'):
        print('LOCK:none')
    else:
        started = d.get('started', '')
        age_hrs = (datetime.datetime.utcnow() - datetime.datetime.fromisoformat(started.replace('Z',''))).total_seconds() / 3600 if started else 999
        print('LOCK:active file=' + d['file'] + ' age_hrs=' + str(round(age_hrs, 2)))
except Exception as e:
    print('LOCK:none error=' + str(e))
"
```

Act on the output:
- `LOCK:none` → proceed to Step 1.
- `LOCK:active … age_hrs=X` where X < 2 → **STOP**. Send Telegram: "⚠️ A session is already active for [file]. Aborting to avoid duplicate order. If this is stale, delete `current-session.json` and retry." Then exit.
- `LOCK:active … age_hrs=X` where X >= 2 → stale lock. Send Telegram: "⚠️ Clearing expired session lock for [file]." Proceed to Step 1.

---

## Step 1 — Download AOI File from Telegram

**SESSION LOCK — write atomically before downloading the AOI:**

```bash
printf '{"file":"%s","started":"%s"}\n' \
  "<FILENAME_FROM_TELEGRAM>" \
  "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  > /home/openclaw/.openclaw/workspace/current-session.json.tmp \
  && mv /home/openclaw/.openclaw/workspace/current-session.json.tmp \
        /home/openclaw/.openclaw/workspace/current-session.json
```

Replace `<FILENAME_FROM_TELEGRAM>` with the exact filename attached to the most recent Telegram message.

    # Get file path from Telegram
    curl -s "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/getFile?file_id=FILE_ID"
    # Returns: { "result": { "file_path": "documents/file_123.geojson" } }

    # Download file
    curl -s -o /home/openclaw/planet_orders/aoi/incoming.geojson \
      "https://api.telegram.org/file/bot${TELEGRAM_BOT_TOKEN}/documents/file_123.geojson"

Convert if needed:
    ogr2ogr -f GeoJSON aoi.geojson input.kml    # KML
    ogr2ogr -f GeoJSON aoi.geojson input.shp    # Shapefile
    # If multi-feature GeoJSON: extract FIRST feature only (always overwrites — no stale AOI)
    MULTI_AOI=$(python3 -c "
import json, sys
raw = json.load(open('/home/openclaw/planet_orders/aoi/incoming.geojson'))
n = 1
if raw.get('type') == 'FeatureCollection' and len(raw['features']) > 1:
    n = len(raw['features'])
    raw['features'] = raw['features'][:1]
json.dump(raw, open('/home/openclaw/planet_orders/aoi/aoi.geojson', 'w'))
print(n)
")
    if [ "${MULTI_AOI:-1}" -gt "1" ] 2>/dev/null; then
      curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
        -d "chat_id=CHAT_ID" \
        -d "text=⚠️ Your file had ${MULTI_AOI} AOIs — only the first one will be ordered."
    fi

**After AOI is saved, send (via curl):**
    curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
      -d "chat_id=CHAT_ID" \
      -d "text=📥 Got your AOI. Uploading to Planet Explorer..."

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

**After AOI polygon is confirmed on map, send (via curl):**
    curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
      -d "chat_id=CHAT_ID" \
      -d "text=🔍 Searching [DATE_FROM] – [DATE_TO]..."

---

## Step 4 — Apply Filters and Search

- Set date range from user message
- Cloud cover <= 20% (default)
- Imagery type: PlanetScope / PSScene, 3-band Visual RGB
- Run search and wait for results

---

## Step 5 — Select Scenes (COST CONTROL + FACILITY CLOUD CHECK)

**SESSION CHECK — before searching scenes:**

```bash
cat /home/openclaw/.openclaw/workspace/current-session.json
```

Confirm the AOI you are about to search scenes for exactly matches the `"file"` field. If it does not match, STOP and send to Telegram: "⚠️ Session mismatch at scene search — aborting."

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

**PRE-ORDER CHECK:**

```bash
cat /home/openclaw/.openclaw/workspace/current-session.json
```

Confirm the order you are about to place is for the file named in the lock. If it does not match, STOP immediately.

Planet charges per clip operation — clipping on Planet's side wastes money. Order **full scenes** instead and clip locally for free. Full scene TIFs are also kept permanently for future re-cropping without re-ordering.

**Order settings:**
- Select chosen scenes
- Order name format: DDMMYYYY_LocationName_Nsc  (e.g. 20022026_IsaTownTest_2sc)
- Bundle: Visual (RGB GeoTIFF)
- **Do NOT enable Clip to AOI** — order full scenes only
- Submit order
- Note the order ID and order URL from the confirmation
- Record the order start time (for elapsed time tracking in Step 7)

**After order is submitted, send (via curl):**
    curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
      -d "chat_id=CHAT_ID" \
      -d "text=🛒 Order placed ([N] scenes). Waiting for Planet to process..."

---

## Step 7 — Poll Until Ready and Download (DO NOT STOP UNTIL DONE)

After ordering, Planet redirects to the order detail page:
  https://insights.planet.com/data/orders/[ORDER-ID]

**Poll this page every 30 seconds. Every 5 minutes, send ONE status message to Telegram (via curl):**

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

**After download is complete, send (via curl):**
    curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
      -d "chat_id=CHAT_ID" \
      -d "text=✂️ Download complete. Mosaicking and clipping..."

**Alternative download via curl using browser cookies:**
If clicking download does not work, extract cookies and use curl:
    # Get cookies via openclaw browser cookies command
    # Then: curl -b "cookie_string" -L "download_url" -o output.zip

**Timeout: 20 minutes.** If exceeded, notify user with the order URL to check manually.

**NEVER go idle while waiting. Poll continuously. Do not wait for the user to check in.**

---

## Step 8 — Local Mosaic + Clip to PNG

Full scenes are stored locally. Mosaic all scenes first (fills any gaps between tiles), then clip to AOI, then convert to PNG. The output PNG is named after the order.

    # Send status before clipping (via curl):
    curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
      -d "chat_id=CHAT_ID" \
      -d "text=🖼️ Converting to PNG..."

    # Mosaic + clip to AOI + convert to PNG via Python helper.
    # Handles: path drift (searches both ~/planet_orders/output/NAME and ~/planet_orders/NAME),
    # zip extraction, recursive TIF find (PSScene/SCENE_ID/ nesting), gdalwarp + gdal_translate.
    if ! PNG_PATH=$(python3 /home/openclaw/planet_orders/clip_order.py \
      "[ORDER_NAME]" \
      /home/openclaw/planet_orders/aoi/aoi.geojson); then
      echo "ERROR: clip_order.py failed — check stderr above"; exit 1
    fi

**Why this approach:**
- `clip_order.py` searches `~/planet_orders/output/NAME/` and `~/planet_orders/NAME/` — handles agent path drift
- Finds TIFs with `rglob` — handles Planet's `PSScene/SCENE_ID/` nested extraction layout
- Single `gdalwarp` call: multiple input scenes + clip in one streaming pass (low RAM)
- No `-dstalpha` — no alpha channel, no black transparent areas at edges
- `-dstnodata "0 0 0"` fills any uncovered edge pixels with black
- Full scenes preserved in `scenes/` for future re-clipping without re-ordering

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

Then send the file (via curl):

    curl -s \
      -F "chat_id=CHAT_ID" \
      -F "document=@/home/openclaw/.openclaw/workspace/[ORDER_NAME].png" \
      -F "caption=ORDER_NAME | DATE | N scenes | CLOUD% cloud | WxH px" \
      "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendDocument"

**Caption rules:** Do NOT include the phrase "clipped to AOI". The caption format is: `[ORDER_NAME] | [DATE] | [N] scenes | [CLOUD]% cloud | [W]x[H] px`

### 9c — Send the catbox download link (MANDATORY)

**This step is required. Do not skip it.**

    curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
      -d "chat_id=CHAT_ID" \
      -d "text=Download (full quality): $CATBOX_URL"

Send this as a separate message immediately after the PNG file. If catbox upload failed in 9a, skip this step only — it does not block the rest of the workflow.

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


**SESSION CLOSE — clear the lock atomically:**

```bash
printf 'null\n' \
  > /home/openclaw/.openclaw/workspace/current-session.json.tmp \
  && mv /home/openclaw/.openclaw/workspace/current-session.json.tmp \
        /home/openclaw/.openclaw/workspace/current-session.json
```


## Step 11 — Confirm and Go Idle

Send summary: order name, date, scene cloud %, facility cloud %, scenes, file size, dimensions.
Then go fully idle.

---

## Crop Mode (any time)

User sends "Crop last image to [coords]" or attaches a sub-AOI. Use the stored full scenes in `~/planet_orders/[ORDER_NAME]/scenes/` — no need to re-order:

    # Re-clip stored scenes to new sub-AOI via Python helper:
    python3 /home/openclaw/planet_orders/clip_order.py \
      "[ORDER_NAME]" \
      /path/to/sub-aoi.geojson

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
