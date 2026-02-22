---
name: planet-order
description: "Order satellite imagery from Planet Explorer via browser automation. Handles multi-scene AOI coverage, mosaics scenes into one PNG, sends via Telegram. Trigger: user sends a location name, AOI file (.geojson/.kml/.shp), or asks for Planet imagery for a location + date range."
metadata: {"openclaw":{"requires":{"bins":["gdal_translate","gdalwarp","gdal_merge.py"]},"emoji":"🛰️"}}
---

# Planet Satellite Imagery Order Skill

Orders the minimum set of Planet scenes needed to cover an AOI, mosaics them into one PNG, sends via Telegram. Goes idle after delivery.

Credentials in environment: PL_EMAIL, PL_PASSWORD, TELEGRAM_BOT_TOKEN.

---

## Triggering

Activate when user:
- Sends an AOI file (.geojson, .kml, .zip shapefile) via Telegram
- Mentions a location name + date range
- Says "order imagery", "get Planet image", "satellite image for [location]"

---

## Saved Locations

| Name | SW (lon, lat) | NE (lon, lat) |
|------|--------------|--------------|
| Isa Town | 50.54, 25.92 | 50.56, 25.94 |
| Isa Bahrain | 50.5, 26.1 | 50.65, 26.25 |

User can add: "Save location [name] at [coordinates or upload AOI]"

---

## Input Parsing

From user message extract:
- **AOI**: file attachment (any format) OR named location OR coordinates
- **Date range**: start and end dates (default: last 30 days if not given)
- **Cloud cover max**: default 20% (be generous — if too strict, no results)

If AOI is a file, save to `~/planet_orders/aoi/[name].[ext]` and convert to GeoJSON for processing.

---

## Step 1 — Convert AOI to GeoJSON

If user sends a file:
```bash
# Convert KML → GeoJSON
ogr2ogr -f GeoJSON aoi.geojson input.kml
# Convert Shapefile → GeoJSON
ogr2ogr -f GeoJSON aoi.geojson input.shp
# GeoJSON → use directly
cp input.geojson aoi.geojson
```

Extract bounding box from GeoJSON for Planet search.

---

## Step 2 — Login to Planet Explorer

- Navigate to https://www.planet.com/explorer/
- If not logged in: click "Sign In", enter PL_EMAIL + PL_PASSWORD from env
- Wait 10 seconds — heavy React SPA
- Confirm login succeeded before continuing

---

## Step 3 — Search for Scenes

- Draw or set AOI on map (use coordinates from Step 1)
- Set date range and cloud cover filter
- Select: PlanetScope / PSScene, 3-band Visual (RGB)
- Run search, wait for results to load

---

## Step 4 — Smart Scene Selection (COST CONTROL)

**Goal: minimum scenes to fully cover the AOI, from the best single day.**

1. **Group all results by acquisition date**
2. **For each date**, check: do the scenes collectively cover ≥ 95% of the AOI?
3. **Rank dates** by:
   - Coverage of AOI (higher = better)
   - Average cloud cover (lower = better)
   - Recency (more recent = better, tiebreaker only)
4. **Pick the best date**
5. **Select only the scenes from that date** that intersect the AOI — no duplicates, no extras
6. **Before ordering**, report to user via Telegram:
   ```
   🛰️ Found coverage for [location]
   📅 Best date: [date]
   🖼️ Scenes needed: [N] (to cover AOI fully)
   ☁️ Avg cloud cover: [X]%
   📦 Ordering now...
   ```

If no single date gives ≥ 95% coverage, pick the date with highest coverage and note the gap.

---

## Step 5 — Place Order

- Click "Order" for each selected scene (or select all then order)
- Order name: `DDMMYYYY_[LocationName]_[N]scenes`
- Bundle: Visual (RGB GeoTIFF)
- Enable: **Clip to AOI** ← critical, reduces file size and cost
- Confirm and submit

---

## Step 6 — Wait and Download

- Go to Orders page
- Poll every 30 seconds until all scenes status = "Success"
- Download all GeoTIFFs to `~/planet_orders/[order_name]/`
- Timeout: 20 minutes

---

## Step 7 — Mosaic and Export PNG

If 1 scene:
```bash
gdal_translate -of PNG -scale input.tif output.png
```

If multiple scenes — merge into one mosaic:
```bash
# Merge scenes into one GeoTIFF
gdal_merge.py -o merged.tif scene1.tif scene2.tif scene3.tif

# Clip to exact AOI bounding box
gdalwarp -cutline aoi.geojson -crop_to_cutline merged.tif clipped.tif

# Convert to PNG
gdal_translate -of PNG -scale clipped.tif output.png
```

Output: `~/planet_orders/[order_name]/[order_name].png`

---

## Step 8 — Send PNG via Telegram

Send the PNG as a Telegram document (supports any file size, any format):
```bash
curl -s \
  -F "chat_id=[USER_CHAT_ID]" \
  -F "document=@output.png" \
  -F "caption=🛰️ [order_name] | [location] | [date] | [N] scenes | ☁️ [X]% cloud" \
  "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendDocument"
```

If PNG > 50MB: split into tiles or reduce resolution first:
```bash
convert -resize 50% output.png output_small.png
```

---

## Step 9 — Confirm and Go Idle

Send summary to Telegram:
- Order name, acquisition date, cloud cover %
- Number of scenes mosaicked
- Output file size
- "Done. Ready for next request."

Then go fully idle. No background polling, no scheduled tasks.

---

## Error Handling

| Error | Action |
|-------|--------|
| No scenes found | Expand cloud cover to 30%, expand date range by 14 days, retry once |
| AOI coverage < 50% | Notify user, ask if they want to proceed with partial coverage |
| Login fails | Notify user on Telegram: "Check Planet credentials" |
| Order fails | Report Planet error message on Telegram |
| Download timeout (>20min) | Notify user, retry once |
| PNG > 50MB | Compress to 50% resolution before sending |
| Mosaic fails | Send individual scene files separately |

