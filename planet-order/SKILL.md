---
name: planet-order
description: "Order satellite imagery from Planet Explorer via browser automation. User sends AOI file (GeoJSON/KML/Shapefile) + date range via Telegram. Skill downloads AOI, orders full scenes, clips locally, sends PNG. Trigger: user sends a file attachment with dates, or mentions a location + date range."
metadata: {"openclaw":{"requires":{"bins":["gdal_translate","gdalwarp","ogr2ogr","curl","unzip","python3"]},"emoji":"🛰️"}}
---

# Planet Satellite Imagery Order Skill

**Flow:** AOI from Telegram → Planet Explorer (browser) → select scenes → order full scenes (Visual bundle, no clip) → poll via Planet API → download zip → extract → `clip_order.py` clips + converts locally → PNG → Telegram.

Credentials in env: `PL_EMAIL`, `PL_PASSWORD`, `PL_API_KEY`, `TELEGRAM_BOT_TOKEN`.

---

## ⚠️ ABSOLUTE CONSTRAINTS

1. **ONE COMMAND FOR CLIPPING — `clip_order.py`.** Never call `gdalwarp`, `gdal_merge`, or `gdal_translate` directly. `clip_order.py` does everything.
2. **NEVER send an image without clipping first.** If `clip_order.py` fails, send an error message and STOP.
3. **ALWAYS write `aoi.geojson` in Step 1.** Verify it exists before proceeding. If missing, STOP.
4. **ONE SESSION AT A TIME.** Check session lock at Step 0. Do not process two AOIs in one run.
5. **ALL Telegram messages via `curl`.** Never use a native OpenClaw message tool.
6. **ONLY the exact stage messages in the Message Reference table are sent.** No narration. No "Let me...", no "Good, ...", no "Now I'll...", no reasoning, no summaries, no status walls. If it's not in the table, don't send it.
7. **ONE SCREENSHOT SENT TO TELEGRAM — AT STEP 5c ONLY.** `openclaw browser screenshot` is a navigation tool used throughout — those are NEVER sent. The ONLY screenshot sent is at Step 5c after the best date is chosen. Steps 1–5b = internal only, never sent.
8. **ALWAYS START FRESH — NEVER ASK ABOUT PREVIOUS ORDERS.** When a GeoJSON + date range is received, begin the workflow immediately. Do NOT check `orders.json`. Do NOT ask "resend or new order?". Previous orders are irrelevant.
9. **NEVER ASK FOR ORDER CONFIRMATION — JUST ORDER.** When AOI cloud ≤ 70%: take screenshot (constraint 7) and place the order. No "proceed?", no "yes/no?", no waiting.
10. **NEW FILE ATTACHMENT = FULL RESET. IGNORE ALL PRIOR CONVERSATION.** When the user sends a file, treat it as a completely new request. All previous bot messages, suggestions, and confirmation states are void. Start Step 0 immediately.
11. **ALWAYS ORDER VISUAL BUNDLE — NEVER ANALYTIC.** In Planet Explorer order dialog, select bundle labeled **"Visual"** or **"Basic RGB"** (3-band GeoTIFF). NEVER select Analytic, Analytic SR, Multispectral, UDM2, or any other bundle. If uncertain, look for the option that says "Visual" explicitly.
12. **NEVER WRITE CUSTOM SCRIPTS.** Do not create `poll_and_process.py`, `download.py`, or any other Python/shell scripts. Use only the commands shown in this SKILL.md. Use `clip_order.py` for all local processing.
13. **POLL AND DOWNLOAD VIA PLANET API — NOT THE BROWSER.** After placing the order, use `curl` with `PL_API_KEY` to poll and download. Do not use the browser to find a Download button.

---

## Telegram Message Reference

These are the ONLY messages sent to Telegram. Nothing else, ever.

| Stage | Exact message |
|-------|--------------|
| AOI saved | `📥 Got your AOI. Uploading to Planet Explorer...` |
| Searching | `🔍 Searching [DATE_FROM] – [DATE_TO]...` |
| Pre-order | See Step 5c — one sendPhoto |
| Order placed | `🛒 Order placed ([N] scenes). Waiting for Planet to process...` |
| Every 5 min | `⏳ Order processing... [X] min elapsed. Planet usually takes 15–30 min.` |
| Download done | `✂️ Download complete. Clipping to AOI...` |
| Clipping done | `🖼️ Converting to PNG...` |
| Done | Send PNG file + caption |
| After PNG | `Download (full quality): [CATBOX_URL]` |
| Any error | One-line error message, then STOP |

---

## Triggering

- User sends `.geojson`, `.kml`, or `.zip` → always a Planet order
- User says "order / get image / satellite" + location + date → order
- User says "crop last image" → Crop Mode (bottom of this doc)
- GeoJSON received with no date → ask for date range only, then proceed immediately
- GeoJSON received with date range → start immediately, no questions, no order history lookup

---

## Saved Locations

| Name | SW (lon, lat) | NE (lon, lat) |
|------|--------------|--------------|
| Isa Town | 50.54, 25.92 | 50.56, 25.94 |
| Isa Bahrain | 50.5, 26.1 | 50.65, 26.25 |

---

## Step 0 — Session Lock Check

```bash
python3 -c "
import json, sys, os, datetime
f = '/home/openclaw/.openclaw/workspace/current-session.json'
if not os.path.exists(f):
    print('LOCK:none'); sys.exit(0)
try:
    d = json.load(open(f))
    if not isinstance(d, dict) or not d.get('file'):
        print('LOCK:none')
    else:
        age = (datetime.datetime.utcnow() - datetime.datetime.fromisoformat(d['started'].replace('Z',''))).total_seconds()/3600
        print('LOCK:active file=' + d['file'] + ' age_hrs=' + str(round(age,2)))
except:
    print('LOCK:none')
"
```

- `LOCK:none` → proceed to Step 1
- `LOCK:active age_hrs < 2` → STOP. Send: `"⚠️ Session already active for [file]. Delete current-session.json to reset."`
- `LOCK:active age_hrs >= 2` → stale. Send: `"⚠️ Clearing expired lock for [file]."` Proceed to Step 1.

---

## Step 1 — Download AOI + Write Canonical File

### 1a — Write session lock

```bash
printf '{"file":"%s","started":"%s"}\n' \
  "FILENAME_FROM_TELEGRAM" \
  "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  > /home/openclaw/.openclaw/workspace/current-session.json.tmp \
  && mv /home/openclaw/.openclaw/workspace/current-session.json.tmp \
        /home/openclaw/.openclaw/workspace/current-session.json
```

### 1b — Download the file

```bash
# Get file_path from Telegram:
curl -s "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/getFile?file_id=FILE_ID"

# Download to incoming:
curl -s -o /home/openclaw/planet_orders/aoi/incoming.geojson \
  "https://api.telegram.org/file/bot${TELEGRAM_BOT_TOKEN}/PATH_FROM_ABOVE"
```

For KML: `ogr2ogr -f GeoJSON /home/openclaw/planet_orders/aoi/incoming.geojson input.kml`
For SHP: `ogr2ogr -f GeoJSON /home/openclaw/planet_orders/aoi/incoming.geojson input.shp`

### 1c — Write canonical aoi.geojson ← MANDATORY, NO SKIPPING

```bash
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
```

If multi-feature, send: `"⚠️ Your file had ${MULTI_AOI} AOIs — only the first one will be ordered."`

### 1d — Verify

```bash
[ -f /home/openclaw/planet_orders/aoi/aoi.geojson ] || {
  curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
    -d "chat_id=CHAT_ID" -d "text=❌ Failed to save AOI. Aborting."
  exit 1
}
```

### 1e — Confirm to user

```bash
curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
  -d "chat_id=CHAT_ID" \
  -d "text=📥 Got your AOI. Uploading to Planet Explorer..."
```

---

## Step 2 — Login to Planet Explorer

1. Open `https://www.planet.com/explorer/` in headless browser
2. Sign in with `PL_EMAIL` / `PL_PASSWORD` from env if not already logged in
3. Wait 10 seconds for the SPA to load
4. Confirm login before proceeding

---

## Step 3 — Upload AOI

1. Find AOI upload button (folder icon in map toolbar near draw tools)
2. Upload: `openclaw browser upload /home/openclaw/planet_orders/aoi/aoi.geojson`
3. Wait for AOI polygon to appear on map
4. Fallback: draw bounding box manually from AOI coordinates

```bash
curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
  -d "chat_id=CHAT_ID" \
  -d "text=🔍 Searching [DATE_FROM] – [DATE_TO]..."
```

---

## Step 4 — Apply Filters and Search

- Date range: from user message
- Cloud cover: ≤ 20% (expand to 50% if no results)
- Imagery type: PlanetScope / PSScene, 3-band Visual RGB
- Run search, wait for results

---

## Step 5 — Pick Best Date

**Selection logic:**
1. Only scenes that intersect the AOI geometry
2. Group by date → find minimum scenes covering ≥ 90% AOI
3. Rank: coverage % desc → scene cloud % asc → most recent
4. Pick the top date

### 5b — Cloud check (read from UI — do NOT use the map viewport)

**The map viewport on the right does NOT render in headless Chrome. Ignore it. All cloud data is in the left panel.**

For the top 1–2 candidate dates in the Daily Results panel:
1. Read the cloud % shown in the scene row (e.g. "28.66%", "100%")
2. Click the scene row to select it
3. Decision:
   - Cloud % < 40% → ✅ proceed
   - Cloud % 40–70% → ⚠️ try next candidate; if none better, proceed with lowest
   - Cloud % > 70% → ❌ skip, try next candidate
4. If best available date has cloud > 70%, send TEXT warning and wait:
   ```bash
   curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
     -d "chat_id=CHAT_ID" \
     -d "text=⚠️ Best available: [DATE] | [X]% cloud | No clear date found. Proceed? (reply yes/no)"
   ```

### 5c — Pre-order screenshot (the ONE photo sent — here and only here)

Take ONE screenshot of the results panel, send it, then immediately go to Step 6:

```bash
openclaw browser screenshot /tmp/cloud-check.png

curl -s \
  -F "chat_id=CHAT_ID" \
  -F "photo=@/tmp/cloud-check.png" \
  -F "caption=✅ [location] | [DATE] | [N] scenes | [X]% cloud | Ordering now..." \
  "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendPhoto"
```

Do not wait for a reply. Do not take or send any more screenshots. Proceed to Step 6 immediately.

---

## Step 6 — Place Order

**Order settings — get these right or the download will break:**
- Select the scenes from Step 5
- Order name: `DDMMYYYY_LocationName_Nsc` (e.g. `20072025_BerlinAirport_3sc`)
- Bundle: **Visual** (3-band RGB GeoTIFF) — look for the option explicitly labeled "Visual" or "Basic RGB". NEVER choose Analytic, Analytic SR, Multispectral, or UDM2.
- **DO NOT enable "Clip to AOI"** — order full scenes, clip locally with `clip_order.py`
- Submit and note the order ID from the confirmation page or API response

```bash
curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
  -d "chat_id=CHAT_ID" \
  -d "text=🛒 Order placed ([N] scenes). Waiting for Planet to process..."
```

---

## Step 7 — Poll via Planet API + Download

**Use the Planet API directly — do NOT use the browser to find a Download button. Do NOT write a custom Python script.**

```bash
ORDER_ID="[ORDER_ID_FROM_STEP_6]"
ORDER_NAME="[ORDER_NAME]"
ORDER_DIR="/home/openclaw/planet_orders/output/${ORDER_NAME}"
mkdir -p "$ORDER_DIR"

START_TIME=$(date +%s)
LAST_STATUS_TIME=$START_TIME

while true; do
  STATE=$(curl -s -u "${PL_API_KEY}:" \
    "https://api.planet.com/compute/ops/orders/v2/${ORDER_ID}" \
    | python3 -c "import json,sys; print(json.load(sys.stdin).get('state','unknown'))")

  ELAPSED=$(( ($(date +%s) - START_TIME) / 60 ))

  if [ "$STATE" = "success" ]; then
    echo "Order complete"
    break
  elif [ "$STATE" = "failed" ] || [ "$STATE" = "cancelled" ]; then
    curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
      -d "chat_id=CHAT_ID" -d "text=❌ Order ${STATE}. Please try again."
    exit 1
  fi

  # Send Stage 5 every 5 minutes
  NOW=$(date +%s)
  if [ $(( NOW - LAST_STATUS_TIME )) -ge 300 ]; then
    curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
      -d "chat_id=CHAT_ID" \
      -d "text=⏳ Order processing... ${ELAPSED} min elapsed. Planet usually takes 15–30 min."
    LAST_STATUS_TIME=$NOW
  fi

  # Timeout after 20 minutes
  if [ $ELAPSED -ge 20 ]; then
    curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
      -d "chat_id=CHAT_ID" \
      -d "text=⏰ Order taking longer than 20 min. Check: https://insights.planet.com/data/orders/${ORDER_ID}"
    exit 1
  fi

  sleep 30
done
```

**Download the zip when order is complete:**

```bash
# Get the zip download URL from the order results
ZIP_URL=$(curl -s -u "${PL_API_KEY}:" \
  "https://api.planet.com/compute/ops/orders/v2/${ORDER_ID}" \
  | python3 -c "
import json, sys
r = json.load(sys.stdin)
for item in r.get('_links', {}).get('results', []):
    if item.get('name', '').endswith('.zip'):
        print(item['location'])
        break
")

if [ -z "$ZIP_URL" ]; then
  curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
    -d "chat_id=CHAT_ID" -d "text=❌ No zip file found in order results."
  exit 1
fi

# Download the zip
curl -s -L "$ZIP_URL" -o "${ORDER_DIR}/order.zip"

curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
  -d "chat_id=CHAT_ID" \
  -d "text=✂️ Download complete. Clipping to AOI..."
```

---

## Step 8 — Clip to AOI

⚠️ **USE `clip_order.py` ONLY. NEVER call `gdalwarp`, `gdal_merge`, or `gdal_translate` directly. IF `clip_order.py` FAILS → send error, STOP.**

```bash
curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
  -d "chat_id=CHAT_ID" \
  -d "text=🖼️ Converting to PNG..."

if ! PNG_PATH=$(python3 /home/openclaw/planet_orders/clip_order.py \
    "${ORDER_NAME}" \
    /home/openclaw/planet_orders/aoi/aoi.geojson); then
  curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
    -d "chat_id=CHAT_ID" \
    -d "text=❌ Clipping failed. No image sent. Check server logs."
  exit 1
fi
```

`clip_order.py` handles automatically:
- Finding TIFs in any subdirectory of `~/planet_orders/output/[ORDER_NAME]/`
- Zip extraction if needed
- Invalid AOI geometry (`-makevalid`)
- Mosaic (`gdalwarp`) + PNG conversion (`gdal_translate`)

Output: `/home/openclaw/planet_orders/output/[ORDER_NAME].png`

---

## Step 9 — Send via Telegram

**Never send a PNG older than the current order start time.**

### 9a — Upload to catbox.moe

```bash
CATBOX_URL=$(curl -4 -s \
  -F "fileToUpload=@/home/openclaw/planet_orders/output/${ORDER_NAME}.png" \
  -F "reqtype=fileupload" \
  https://catbox.moe/user/api.php)
```

### 9b — Send PNG

```bash
cp /home/openclaw/planet_orders/output/${ORDER_NAME}.png \
   /home/openclaw/.openclaw/workspace/${ORDER_NAME}.png

curl -s \
  -F "chat_id=CHAT_ID" \
  -F "document=@/home/openclaw/.openclaw/workspace/${ORDER_NAME}.png" \
  -F "caption=${ORDER_NAME} | [DATE] | [N] scenes | [CLOUD]% cloud | [W]x[H] px" \
  "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendDocument"
```

Caption format: `ORDER_NAME | DATE | N scenes | X% cloud | WxH px`
Do NOT say "clipped to AOI" in the caption.

### 9c — Send catbox link (mandatory)

```bash
curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
  -d "chat_id=CHAT_ID" \
  -d "text=Download (full quality): ${CATBOX_URL}"
```

If PNG > 50MB, compress first:
```bash
convert -resize 50% /home/openclaw/planet_orders/output/${ORDER_NAME}.png \
                    /home/openclaw/planet_orders/output/${ORDER_NAME}.png
```

---

## Step 10 — Log + Close Session

```bash
python3 -c "
import json, os
log = '/home/openclaw/planet_orders/orders.json'
entries = json.load(open(log)) if os.path.exists(log) else []
entries.append({
    'date': 'IMAGERY_DATE',
    'ordered_at': 'TIMESTAMP',
    'name': 'ORDER_NAME',
    'location': 'LOCATION_NAME',
    'scenes': N,
    'cloud_pct': X,
    'size_mb': FILESIZE_MB,
    'catbox_url': 'CATBOX_URL',
    'sent': True
})
json.dump(entries, open(log, 'w'), indent=2)
"

printf 'null\n' \
  > /home/openclaw/.openclaw/workspace/current-session.json.tmp \
  && mv /home/openclaw/.openclaw/workspace/current-session.json.tmp \
        /home/openclaw/.openclaw/workspace/current-session.json
```

---

## Step 11 — Confirm and Go Idle

Send one summary message: order name, date, scene cloud %, scenes, file size, dimensions.
Then go fully idle.

---

## Crop Mode

User says "crop last image to [coords]" or sends a sub-AOI file. Stored TIFs are in `~/planet_orders/output/[ORDER_NAME]/` — no re-order needed.

```bash
python3 /home/openclaw/planet_orders/clip_order.py \
  "[ORDER_NAME]" \
  /path/to/sub-aoi.geojson
```

---

## Error Handling

| Error | Action |
|-------|--------|
| AOI upload fails | Draw bounding box manually from AOI coordinates |
| No scenes found | Expand cloud to 50%, extend date ±14 days, retry once |
| Coverage < 90% | Notify user, report %, ask to proceed |
| Login fails | Notify user to check credentials, stop |
| Order state = failed/cancelled | Send error message, stop |
| Order timeout > 20 min | Send order URL, ask user to check manually |
| No zip in order results | Send error, stop |
| `clip_order.py` fails | Send error to Telegram, STOP — never send unclipped image |
| PNG > 50MB | Compress 50%, resend |
