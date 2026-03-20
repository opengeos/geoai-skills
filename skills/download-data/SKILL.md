---
name: download-data
description: >
  Download NAIP aerial imagery for a bounding box. Specify coordinates as
  minx,miny,maxx,maxy in WGS84 and optionally a year.
argument-hint: <minx,miny,maxx,maxy> [--year YYYY] [--output DIR] [--max-items N]
allowed-tools: Bash
---

You are helping the user download NAIP aerial imagery using geoai.

Input: `$@`

Follow these steps in order.

## Step 1 -- Parse arguments

Extract the bounding box from the first argument (comma-separated `minx,miny,maxx,maxy`).

Parse optional flags from remaining arguments:
- `--year YYYY` -> download year (default: most recent available)
- `--output DIR` -> output directory (default: `./naip_data/`)
- `--max-items N` -> maximum number of items to download (default: 10)

If the input is natural language (e.g. "download NAIP imagery for Knoxville, TN"), extract or infer the bounding box. If you cannot determine the bbox, ask the user for coordinates.

## Step 2 -- Validate the bounding box

Confirm the bounding box has 4 numeric values and represents a valid geographic extent:
- `minx < maxx` and `miny < maxy`
- Longitude values within -180 to 180
- Latitude values within -90 to 90
- The area is not unreasonably large (warn if the bbox spans more than 1 degree in either direction)

If validation fails, report the issue and ask for corrected coordinates.

## Step 3 -- Run the download

```bash
python3 -c "
import geoai, os

bbox = (MINX, MINY, MAXX, MAXY)
output_dir = 'OUTPUT_DIR'
os.makedirs(output_dir, exist_ok=True)

result = geoai.download_naip(
    bbox=bbox,
    output_dir=output_dir,
    year=YEAR,
    max_items=MAX_ITEMS,
)
if isinstance(result, list):
    for f in result:
        size_mb = os.path.getsize(f) / (1024 * 1024) if os.path.exists(f) else 0
        print(f'{f} ({size_mb:.1f} MB)')
    print(f'Total files: {len(result)}')
elif isinstance(result, str):
    size_mb = os.path.getsize(result) / (1024 * 1024) if os.path.exists(result) else 0
    print(f'{result} ({size_mb:.1f} MB)')
else:
    print(f'Result: {result}')
"
```

Replace `MINX`, `MINY`, `MAXX`, `MAXY`, `OUTPUT_DIR`, `YEAR`, and `MAX_ITEMS` with actual values.

For the year parameter:
- If `--year` was specified, use that value (e.g. `year=2022`)
- If not specified, omit the parameter or pass `year=None` to get the most recent available

## Step 4 -- Update state

If a state directory exists, update it with the downloaded file paths:

```bash
STATE_DIR=""
test -f .geoai-skills/state.json && STATE_DIR=".geoai-skills"
PROJECT_ROOT="$(git rev-parse --show-toplevel 2>/dev/null || echo "$PWD")"
PROJECT_ID="$(echo "$PROJECT_ROOT" | tr '/' '-')"
test -f "$HOME/.geoai-skills/$PROJECT_ID/state.json" && STATE_DIR="$HOME/.geoai-skills/$PROJECT_ID"
```

If `STATE_DIR` is set:

```bash
python3 -c "
import json, os
state_file = 'STATE_DIR/state.json'
state = {}
if os.path.exists(state_file):
    with open(state_file) as f:
        state = json.load(f)
state.setdefault('downloaded_files', [])
state['downloaded_files'].extend(DOWNLOADED_FILES)
with open(state_file, 'w') as f:
    json.dump(state, f, indent=2)
"
```

## Step 5 -- Report results

Summarize the download:
- Number of files downloaded
- File paths and sizes
- Coverage area (bounding box)
- Year of imagery

Then suggest: *"Use `/geoai-skills:inspect-geo` to examine the downloaded imagery, or `/geoai-skills:detect-objects` to run AI models on it."*

## Error handling

- **`import geoai` fails** -> delegate to `/geoai-skills:install-geoai`.
- **Network error** -> report the error and suggest retrying.
- **No data available** for the specified region/year -> suggest trying a different year or expanding the bounding box.
- **Timeout** -> suggest reducing `--max-items` or using a smaller bounding box.
