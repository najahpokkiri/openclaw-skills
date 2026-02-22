---
name: planet-order
description: "Order satellite imagery from Planet Explorer via browser automation. User sends AOI file (GeoJSON/KML/Shapefile) + date range via Telegram. Skill downloads AOI, uploads it to Planet Explorer, selects minimum scenes covering the AOI, orders clipped delivery, and sends PNG via Telegram. Trigger: user sends a file attachment with dates, or mentions a saved location + date range."
metadata: {"openclaw":{"requires":{"bins":["gdal_translate","gdalwarp","ogr2ogr","curl"]},"emoji":"🛰️"}}
---

# Planet Satellite Imagery Order Skill

Receives AOI from Telegram → uploads to Planet Explorer → selects right scenes → orders → delivers PNG via Telegram.

Credentials in env: PL_EMAIL, PL_PASSWORD, TELEGRAM_BOT_TOKEN.

---

## Key Principle: Tight AOI = Lower Cost

Small, precise AOI = fewer scenes = lower cost. Planet clips delivered imagery to your AOI boundary.

---

## Triggering

- User sends a file (.geojson, .kml, .zip shapefile) + date range via Telegram
- User mentions a saved location + date range (no file needed)
- "Order imagery for [location] from [date] to [date]"
- "Crop last image to [coords]" — crop-only mode on already-downloaded file

Extract from the message:
- File attachment (if sent) OR location name
- Start date, end date
- Cloud cover max (default 20% if not specified)

---

## Saved Locations

| Name | SW (lon, lat) | NE (lon, lat) |
|------|--------------|--------------|
| Isa Town | 50.54, 25.92 | 50.56, 25.94 |
| Isa Bahrain | 50.5, 26.1 | 50.65, 26.25 |

User can add: "Save location [name] at [coordinates or upload AOI]"

---

## Step 1 — Download AOI File from Telegram

When user sends a file attachment via Telegram, OpenClaw passes the message with a file_id.
Download it using the Telegram Bot API:

    # 1. Get the file path from Telegram
    curl -s "https://api.telegram.org/bot/getFile?file_id=FILE_ID_HERE"
    # Returns: { "result": { "file_path": "documents/file_123.geojson" } }

    # 2. Download the actual file
    curl -s -o /home/openclaw/planet_orders/aoi/incoming.geojson       "https://api.telegram.org/file/bot/documents/file_123.geojson"

Save to: ~/planet_orders/aoi/[location_name].geojson

Convert to GeoJSON if needed:

    ogr2ogr -f GeoJSON aoi.geojson input.kml      # KML
    ogr2ogr -f GeoJSON aoi.geojson input.shp      # Shapefile from .zip (extract first)
    # GeoJSON: use directly

If user gave a saved location name instead of a file, create a bounding box GeoJSON from saved coordinates.

---

## Step 2 — Login to Planet Explorer

- Open https://www.planet.com/explorer/ in headless browser
- Click Sign In if not logged in
- Enter PL_EMAIL, PL_PASSWORD from env
- Wait 10 seconds (heavy React SPA)
- Confirm you are on the explorer map before continuing

---

## Step 3 — Upload AOI File to Planet Explorer

Planet Explorer has an AOI import button — use it to upload the GeoJSON directly:

1. Look for the AOI/draw toolbar on the map (top-left or right panel)
2. Find the "Upload" or "Import AOI" button (folder icon or upload icon near the draw tools)
3. Click it — it opens a file input dialog
4. Use Playwright to set the file on the input element:

       const [fileChooser] = await Promise.all([
         page.waitForEvent('filechooser'),
         page.click('[data-testid="upload-aoi"]'  // or the upload button selector)
       ]);
       await fileChooser.setFiles('/home/openclaw/planet_orders/aoi/aoi.geojson');

5. Wait for the AOI to appear on the map (polygon outline visible)
6. Confirm the AOI boundary matches what was sent

If the file chooser approach does not work, fall back to drawing the bounding box manually using the draw rectangle tool and the AOI's bounding box coordinates.

---

## Step 4 — Apply Filters and Search

- Set date range from user message
- Cloud cover <= max (default 20%)
- Imagery type: PlanetScope / PSScene
- Product: 3-band Visual (RGB)
- Run search and wait for results

---

## Step 5 — Select Only the Right Scenes (COST CONTROL)

**Goal: fewest scenes from the best single day that cover >= 90% of the AOI.**

1. Only consider scenes that actually overlap the AOI geometry (not just bounding box)
2. Group scenes by acquisition date
3. For each date, check which scenes together cover >= 90% of AOI
4. Rank dates: coverage % (desc) → avg cloud cover (asc) → recency (tiebreaker)
5. Select minimum scenes from best date — skip any scene whose area is already covered
6. Notify user via Telegram before ordering:

       Found coverage for [location]
       Best date: [YYYY-MM-DD]
       Scenes to order: [N] (covers [X]% of AOI)
       Avg cloud cover: [Y]%
       Ordering now...

If no date gives >= 90%: notify user, report best available %, ask to proceed.

---

## Step 6 — Place Order

- Select chosen scenes in Planet Explorer
- Order name: DDMMYYYY_LocationName_Nsc
- Bundle: Visual (RGB GeoTIFF)
- **Enable: Clip to AOI** — this is critical, Planet clips delivery to your uploaded boundary
- Submit order

---

## Step 7 — Wait and Download

- Go to Orders page, poll every 30s until status = Success
- Download all delivered GeoTIFFs to ~/planet_orders/[order_name]/
- Timeout: 20 minutes — if exceeded, notify user with order page link

---

## Step 8 — Process to PNG

Planet delivers files already clipped to the AOI boundary.

If 1 scene:

    gdal_translate -of PNG -scale input.tif output.png

If multiple scenes (mosaic):

    gdalwarp -of GTiff scene1.tif scene2.tif merged.tif
    gdal_translate -of PNG -scale merged.tif output.png

### Crop to Sub-Area (Optional)

User can send "Crop last image to [coords]" or attach a sub-AOI GeoJSON at any time.

Crop by bounding box:

    gdal_translate -projwin <ulx> <uly> <lrx> <lry> -of PNG -scale input.tif cropped.png

Crop by polygon:

    gdalwarp -cutline sub_aoi.geojson -crop_to_cutline input.tif clipped.tif
    gdal_translate -of PNG -scale clipped.tif cropped.png

---

## Step 9 — Send PNG via Telegram

    curl -s       -F "chat_id=CHAT_ID"       -F "document=@output.png"       -F "caption=ORDER_NAME | DATE | N scenes | CLOUD% cloud | WxH px"       "https://api.telegram.org/bot/sendDocument"

If PNG > 50MB: compress first with ImageMagick: convert -resize 50% output.png output.png

---

## Step 10 — Confirm and Go Idle

Send summary: order name, date, cloud cover, scenes, file size, pixel dimensions.
"Done. Send another AOI or crop request anytime."
Go fully idle — no background tasks.

---

## Error Handling

- File download from Telegram fails: ask user to resend the file
- AOI upload to Planet fails: fall back to drawing bounding box from AOI coordinates manually
- No scenes found: expand cloud cover to 30%, extend date range +/-14 days, retry once
- Coverage < 90%: notify user, report best %, ask to proceed
- Login fails: notify user to check Planet credentials in .env
- Order timeout > 20min: notify user with order link to check manually
- PNG > 50MB: compress to 50% resolution and resend
