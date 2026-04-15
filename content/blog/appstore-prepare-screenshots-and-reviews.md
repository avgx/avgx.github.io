+++
title = "App Store screenshots and previews without a designer"
description = "Simulator captures, resize png with sips, resize mov with ffmpeg."
date = 2026-04-14

[taxonomies]
tags = ["howto", "publish", "app-store-connect"]
+++

When nobody owns marketing visuals, you can still ship: capture the UI in the Simulator, treat App Store Connect as a checklist of pixel sizes, and re-encode preview video and re-size screenshots.

The goal is to get **accepted media**.

## App Store Connect Requirements for screenshots

Up to 10 screenshots. 

**Phone 6.5" Display**
| Orientation | Value |
| --- | --- |
| Portrait | 1242×2688px or 1284×2778px |
| Landscape | 2688×1242px or 2778×1284px |

 
**iPad 13" Display**
| Orientation | Value |
| --- | --- |
| Portrait | 2064×2752px or 2048×2732px |
| Landscape | 2752×2064px or 2732×2048px |

[Screenshot specifications](https://developer.apple.com/help/app-store-connect/reference/app-information/screenshot-specifications/)

## App Store Connect Requirements for previews

Up to 3 app previews. 

Connect is strict about **frame size** and **audio**. A straight Simulator recording often fails validation; normalize once against ([App preview specifications](https://developer.apple.com/help/app-store-connect/reference/app-information/app-preview-specifications/)).

| Requirement | Value |
| --- | --- |
| Maximum file size | 500 MB |
| Minimum length | 15 seconds |
| Maximum length | 30 seconds |
| Phone (portrait) | 886×1920 px |
| Phone (landscape) | 1920×886 px |
| iPad (portrait) | 1200×1600 px |
| iPad (landscape) | 1600×1200 px |

## Screenshots

Capture with the Simulator device via `⌘S` (*File → Save Screen*).

**Phone** Portrait **1284 × 2778**:

```bash
sips -z 2778 1284 portrait.png --out portrait-1284x2778.png
```

**Phone** Landscape **2778 × 1284**:

```bash
sips -z 1284 2778 landscape.png --out landscape-2778x1284.png
```

**iPad** Portrait **2048 × 2732**:

```bash
sips -z 2732 2048 portrait.png --out portrait-2048x2732.png
```

**iPad** Landscape **2732 × 2048**:

```bash
sips -z 2048 2732 landscape.png --out landscape-2688x1242.png
```

## App Previews

Capture with the Simulator device via `⌘R` (*File → Record Screen*).

```bash
#!/bin/bash

# make_appstore_preview_phone.sh recording.mov
INPUT="$1"
OUTPUT="${INPUT%.*}_preview.mov"

ffmpeg -i "$INPUT" \
    -t 30 -f lavfi -i anullsrc=channel_layout=stereo:sample_rate=44100 \
    -vf "scale=886:1920:force_original_aspect_ratio=decrease,pad=886:1920:(886-iw)/2:(1920-ih)/2" \
    -c:v libx264 -crf 20 -preset faster \
    -c:a aac -b:a 128k \
    -shortest \
    "$OUTPUT" -y

echo "Done: $OUTPUT"
```
