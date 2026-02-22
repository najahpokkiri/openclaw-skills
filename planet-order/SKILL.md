---
name: planet-order
description: "Order satellite imagery from Planet Explorer via browser automation. Selects only scenes covering the specific AOI (not surrounding areas), clips to AOI, optionally crops to sub-area, sends PNG via Telegram. Trigger: user sends AOI file or location + date range."
metadata: {"openclaw":{"requires":{"bins":["gdal_translate","gdalwarp","ogr2ogr"]},"emoji":"🛰️"}}
---

# Planet Satellite Imagery Order Skill

Orders only the scenes needed to cover a specific AOI, clips delivery to that AOI, crops further if requested, sends PNG via Telegram. Goes idle after.

Credentials in env: PL_EMAIL, PL_PASSWORD, TELEGRAM_BOT_TOKEN.

---

## Key Principle: Tight AOI = Lower Cost

The AOI should be drawn tightly around exactly what is needed (e.g. just the airport terminal, not the surrounding farmland). Planet clips delivered imagery to this boundary. A smaller, precise AOI means:
- Fewer scenes needed — scenes are only selected if they actually cover the AOI geometry, not just the bounding box
- Smaller delivered files, lower cost
- No irrelevant coverage (no "green stuff around it")

---

## Triggering

- User sends an AOI file (.geojson, .kml, .zip shapefile) via Telegram
- User mentions a location name + date range
- "Order imagery", "get Planet image", "satellite image for [location]"
- "Crop last image to [coords]" — triggers crop-only on already-downloaded file

---

## Saved Locations

| Name | SW (lon, lat) | NE (lon, lat) |
|------|--------------|--------------|
| Isa Town | 50.54, 25.92 | 50.56, 25.94 |
| Isa Bahrain | 50.5, 26.1 | 50.65, 26.25 |

User can add: "Save location [name] at [coordinates or upload AOI]"

---

## Step 1 — Convert AOI to GeoJSON

If user sends a file:

    ogr2ogr -f GeoJSON aoi.geojson input.kml     # KML
    ogr2ogr -f GeoJSON aoi.geojson input.shp     # Shapefile
    cp input.geojson aoi.geojson                  # GeoJSON direct

Save to ~/planet_orders/aoi/[name].geojson

---

## Step 2 — Login to Planet Explorer

- Navigate to https://www.planet.com/explorer/
- Click Sign In if not logged in, use PL_EMAIL + PL_PASSWORD from env
- Wait 10 seconds (heavy React SPA)
- Confirm login before continuing

---

## Step 3 — Search and Filter Scenes

- Set AOI on map using exact geometry from Step 1
- Apply filters: date range, cloud cover <= max (default 20%), PlanetScope PSScene, 3-band Visual RGB
- Run search and wait for results

---

## Step 4 — Select Only the Right Scenes (COST CONTROL)

**Goal: fewest scenes covering the AOI from the best single day. Never order scenes that only cover surrounding areas.**

1. Only consider scenes that actually overlap the AOI geometry
2. Group scenes by acquisition date
3. For each date, find scenes that together cover >= 90% of the AOI
4. Rank dates: coverage % (high to low) -> avg cloud cover (low to high) -> recency (tiebreaker)
5. From the best date, select only the minimum scenes needed — skip any scene whose area is already covered by another selected scene
6. Before ordering, report to user via Telegram:

    Found coverage for [location]
    Best date: [YYYY-MM-DD]
    Scenes to order: [N] (covers [X]% of AOI)
    Avg cloud cover: [Y]%
    Ordering now...

If no date gives >= 90% coverage: notify user, report best available %, ask to proceed.

---

## Step 5 — Place Order

- Select the chosen scenes in Planet Explorer
- Order name: DDMMYYYY_LocationName_Nsc
- Bundle: Visual (RGB GeoTIFF)
- Enable: Clip to AOI (Planet clips delivery to your boundary — no surrounding areas in output)
- Submit order

---

## Step 6 — Wait and Download

- Poll Orders page every 30s until status = Success
- Download all GeoTIFFs to ~/planet_orders/[order_name]/
- Timeout: 20 minutes

---

## Step 7 — Post-Processing

Planet delivers scenes already clipped to the AOI boundary.

If 1 scene:

    gdal_translate -of PNG -scale input.tif output.png

If multiple scenes:

    gdalwarp -of GTiff scene1.tif scene2.tif merged.tif
    gdal_translate -of PNG -scale merged.tif output.png

### Cropping to Sub-Area (Optional)

User can request a crop at any time — either during the order or afterwards:
- "Crop last image to [lon1,lat1,lon2,lat2]"
- "Crop to this area" + attach a GeoJSON file
- "Crop to just the terminal / northern section / etc."

Crop by bounding box:

    gdal_translate -projwin <ulx> <uly> <lrx> <lry> -of PNG -scale input.tif cropped.png

Crop by polygon (e.g. just the terminal building):

    gdalwarp -cutline sub_aoi.geojson -crop_to_cutline input.tif clipped.tif
    gdal_translate -of PNG -scale clipped.tif cropped.png

Output: [order_name]_cropped.png — send via Telegram the same as the main output.

---

## Step 8 — Send PNG via Telegram

    curl -s \
      -F "chat_id=CHAT_ID" \
      -F "document=@output.png" \
      -F "caption=ORDER_NAME | DATE | N scenes | CLOUD% cloud | WxH px" \
      "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendDocument"

If PNG > 50MB: compress first with ImageMagick: convert -resize 50% output.png output.png

---

## Step 9 — Confirm and Go Idle

Send summary: order name, date, cloud cover, scenes ordered, file size, pixel dimensions.
"Done. Send another AOI or crop request anytime."
Go fully idle — no background tasks.

---

## Error Handling

- No scenes found: expand cloud cover to 30%, extend date range +/-14 days, retry once
- Coverage < 90%: notify user, report best available, ask to proceed
- Login fails: notify user to check Planet credentials
- Order fails: report Planet error message
- Timeout > 20min: notify user with order link to check manually
- PNG > 50MB: compress to 50% resolution and resend
