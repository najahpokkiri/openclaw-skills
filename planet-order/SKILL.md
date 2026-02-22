---
name: planet-order
description: "Order satellite imagery from Planet Explorer via browser automation. User sends AOI file (GeoJSON/KML/Shapefile) + date range via Telegram. Skill downloads AOI, uploads it to Planet Explorer, selects minimum scenes covering the AOI, orders clipped delivery, and sends PNG via Telegram. Trigger: user sends a file attachment with dates, or mentions a saved location + date range."
metadata: {"openclaw":{"requires":{"bins":["gdal_translate","gdalwarp","ogr2ogr","curl"]},"emoji":"🛰️"}}
---

# Planet Satellite Imagery Order Skill

Receives AOI from Telegram → uploads to Planet Explorer → selects right scenes → orders → polls until ready → downloads → converts to PNG → sends via Telegram.

Credentials in env: PL_EMAIL, PL_PASSWORD, TELEGRAM_BOT_TOKEN.

---

## Output Rules

- Do NOT send screenshots, status updates, or progress messages to Telegram while working
- Work silently. The user sees nothing until PNG is ready.
- Only 2 messages total:
  1. Pre-order notice: "Found N scenes for [location] on [date], X% cloud. Ordering..."
  2. Final: send the PNG file with caption
- Exception: if waiting >10 min, send ONE "Still processing, Planet order in queue..." message

---

## Key Principle: Tight AOI = Lower Cost

Small, precise AOI = fewer scenes = lower cost. Planet clips delivered imagery to your AOI boundary.

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
    curl -s -o /home/openclaw/planet_orders/aoi/incoming.geojson       "https://api.telegram.org/file/bot${TELEGRAM_BOT_TOKEN}/documents/file_123.geojson"

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

## Step 5 — Select Scenes (COST CONTROL)

**Goal: fewest scenes from best single day covering >= 90% of AOI.**

1. Only scenes that actually intersect AOI geometry
2. Group by date; find minimum scenes for >= 90% coverage
3. Rank: coverage % desc → cloud cover asc → recency
4. Notify user before ordering:

       Found coverage for [location]
       Best date: [YYYY-MM-DD]
       Scenes to order: [N] (covers [X]% of AOI)
       Avg cloud cover: [Y]%
       Ordering now...

---

## Step 6 — Place Order

- Select chosen scenes
- Order name format: DDMMYYYY_LocationName_Nsc  (e.g. 20022026_IsaTownTest_2sc)
- Bundle: Visual (RGB GeoTIFF)
- **Enable: Clip to AOI**
- Submit order
- Note the order ID and order URL from the confirmation

---

## Step 7 — Poll Until Ready and Download (DO NOT STOP UNTIL DONE)

After ordering, Planet redirects to the order detail page:
  https://insights.planet.com/data/orders/[ORDER-ID]

**Poll this page every 30 seconds:**
1. Take a browser snapshot of https://insights.planet.com/data/orders/[ORDER-ID]
2. Look for status indicators: "success", "complete", "ready", "Download", green checkmark
3. Also check https://www.planet.com/account/#/orders for the order list status

**When order shows as complete/success:**
1. Find the Download button or download links on the order page
2. Click Download — Planet provides a zip file or individual GeoTIFF links
3. Use `openclaw browser waitfordownload` or `openclaw browser download` to save the file
4. Files save to: ~/planet_orders/[order_name]/

**Alternative download via curl using browser cookies:**
If clicking download does not work, extract cookies and use curl:
    # Get cookies via openclaw browser cookies command
    # Then: curl -b "cookie_string" -L "download_url" -o output.zip

**Timeout: 20 minutes.** If exceeded, notify user with the order URL to check manually.

**NEVER go idle while waiting. Poll continuously. Do not wait for the user to check in.**

---

## Step 8 — Process to PNG

    # Single scene:
    gdal_translate -of PNG -scale input.tif output.png

    # Multiple scenes (mosaic):
    gdalwarp -of GTiff scene1.tif scene2.tif merged.tif
    gdal_translate -of PNG -scale merged.tif output.png

    # If downloaded as zip:
    unzip order.zip -d ~/planet_orders/[order_name]/

---

## Step 9 — Send PNG via Telegram

    curl -s       -F "chat_id=CHAT_ID"       -F "document=@output.png"       -F "caption=ORDER_NAME | DATE | N scenes | CLOUD% cloud | WxH px"       "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendDocument"

If PNG > 50MB: `convert -resize 50% output.png output.png`

---

## Step 10 — Confirm and Go Idle

Send summary: order name, date, cloud cover, scenes, file size, dimensions.
Then go fully idle.

---

## Crop Mode (any time)

User sends "Crop last image to [coords]" or attaches a sub-AOI:

    # Bounding box crop:
    gdal_translate -projwin <ulx> <uly> <lrx> <lry> -of PNG -scale input.tif cropped.png

    # Polygon crop:
    gdalwarp -cutline sub.geojson -crop_to_cutline input.tif clipped.tif
    gdal_translate -of PNG -scale clipped.tif cropped.png

---

## Error Handling

- AOI upload fails: draw bounding box manually from AOI coordinates
- No scenes found: expand cloud cover to 30%, extend date range +/-14 days, retry once
- Coverage < 90%: notify user, report best %, ask to proceed
- Login fails: notify user to check credentials
- Download button not found: try navigating directly to insights.planet.com/data/orders/[ID]
- Order timeout > 20 min: notify user with order URL
- PNG > 50MB: compress and resend
