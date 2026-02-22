---
name: planet-order
description: "Order satellite imagery from Planet Explorer via browser automation, crop to AOI, export PNG, and send via Telegram. Trigger: user requests satellite imagery, mentions Planet, or asks for an image order for a location."
metadata: {"openclaw":{"requires":{"bins":["chromium","gdal_translate"]},"emoji":"🛰️"}}
---

# Planet Satellite Imagery Order Skill

Single-purpose: order imagery from Planet Explorer, crop to PNG, send file via Telegram, then idle.

Credentials in environment: PL_EMAIL, PL_PASSWORD, TELEGRAM_BOT_TOKEN.

## When to Trigger
- User mentions Planet Explorer / satellite imagery ordering
- Location name + date range for imagery
- "Get latest image for [location]"
- Any reference to PlanetScope, GeoTIFF, or satellite download

## Input Parsing
Extract from user message:
- **Location name**: e.g. "Isa Town"
- **Coordinates** (if given): lat/lon bounding box
- **Date range**: start and end dates
- **Cloud cover max**: default 10% if not specified

## Saved Locations
| Name | SW Corner (lon, lat) | NE Corner (lon, lat) |
|------|---------------------|---------------------|
| Isa Town | 50.54, 25.92 | 50.56, 25.94 |
| Isa Bahrain | 50.5, 26.1 | 50.65, 26.25 |

User can add more: "Save location [name] at [coordinates]"

## Full Workflow

### 1. Login to Planet Explorer
- Navigate to https://www.planet.com/explorer/
- If not logged in, click "Sign In"
- Read PL_EMAIL and PL_PASSWORD from environment
- Enter credentials, submit login form
- Wait 10 seconds — Planet Explorer is a heavy React SPA

### 2. Set Search Area
- Use search bar or map to go to target coordinates
- Use draw tool to outline AOI
- Default test area: SW 50.54°E 25.92°N to NE 50.56°E 25.94°N

### 3. Apply Filters
- Date range: requested dates
- Cloud cover ≤ requested max (default 10%)
- Imagery: PlanetScope / PSScene
- Product: 3-band Visual (RGB)

### 4. Select Best Image
- Lowest cloud cover → highest visible area → most recent
- Click to preview on map

### 5. Place Order
- Click "Order" / add to cart
- Order name: DDMMYYYY_NZZ_LocationName_3m
- Bundle: Visual (RGB GeoTIFF)
- Enable: Clip to AOI
- Confirm and place

### 6. Wait and Download
- Go to Orders page
- Poll every 30s until status = "Success"
- Download GeoTIFF to ~/planet_orders/[order_name]/
- Timeout: 15 minutes

### 7. Crop and Export PNG
```bash
# If already clipped to AOI:
gdal_translate -of PNG -scale input.tif output.png
# If needs cropping:
gdal_translate -projwin <ulx> <uly> <lrx> <lry> -of PNG input.tif output.png
```
Output: ~/planet_orders/[order_name]/[order_name].png

### 8. Send PNG via Telegram
Send the PNG directly as a Telegram file (no email, no external service):
```bash
curl -F "chat_id=<USER_CHAT_ID>" \
     -F "document=@output.png" \
     -F "caption=🛰️ [order_name] | [location] | [date] | cloud: [X]%" \
     "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendDocument"
```

### 9. Confirm on Telegram and Go Idle
- Confirm: order name, acquisition date, cloud cover %, file size
- Done. Wait for next trigger.

## Error Handling
- No imagery found → expand date range by 5 days, retry once
- Login fails → notify user on Telegram to check credentials
- Order fails → report error on Telegram
- File send fails → try sendPhoto endpoint as fallback
- Timeout: 15 minutes max
