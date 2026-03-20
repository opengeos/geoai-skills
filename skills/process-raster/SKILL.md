---
name: process-raster
description: >
  Process raster data: clip by bounding box, stack multiple bands, mosaic
  GeoTIFFs, or convert between raster and vector formats.
argument-hint: <operation> <input> [options]
allowed-tools: Bash
---

You are helping the user process geospatial raster data using geoai.

Input: `$@`

Follow these steps in order.

## Step 1 -- Determine the operation

Parse `$@` to identify the requested operation:

| Operation | Triggers | Required inputs |
|---|---|---|
| `clip` | "clip", "crop", "subset", `--bbox` present | input raster + bbox |
| `stack` | "stack", "combine bands" | list of input rasters |
| `mosaic` | "mosaic", "merge" | input directory or list of rasters |
| `raster-to-vector` | "to vector", "vectorize", "polygonize" | input raster |
| `vector-to-raster` | "to raster", "rasterize", "burn" | input vector + pixel size |

If the operation is unclear from the input, ask the user to specify.

## Step 2 -- Resolve input file(s)

For single-file operations (`clip`, `raster-to-vector`, `vector-to-raster`):

```bash
find "$PWD" -name "INPUT_FILENAME" -not -path '*/.git/*' 2>/dev/null
```

For multi-file operations (`stack`, `mosaic`), if a directory is given:

```bash
find "INPUT_DIR" -name "*.tif" -o -name "*.tiff" 2>/dev/null | sort
```

If the user recently inspected or downloaded a file and did not specify an input, check the state file for context:

```bash
STATE_DIR=""
test -f .geoai-skills/state.json && STATE_DIR=".geoai-skills"
PROJECT_ROOT="$(git rev-parse --show-toplevel 2>/dev/null || echo "$PWD")"
PROJECT_ID="$(echo "$PROJECT_ROOT" | tr '/' '-')"
test -f "$HOME/.geoai-skills/$PROJECT_ID/state.json" && STATE_DIR="$HOME/.geoai-skills/$PROJECT_ID"
```

If state exists, read the last inspected or downloaded file:

```bash
python3 -c "
import json
with open('STATE_DIR/state.json') as f:
    state = json.load(f)
if 'last_inspected' in state:
    print(f'Last inspected: {state[\"last_inspected\"][\"path\"]}')
if 'downloaded_files' in state:
    for f in state['downloaded_files']:
        print(f'Downloaded: {f}')
"
```

## Step 3 -- Execute the operation

### Clip by bounding box

```bash
python3 -c "
import geoai

result = geoai.clip_raster_by_bbox(
    input_raster='INPUT_PATH',
    output_raster='OUTPUT_PATH',
    bbox=[MINX, MINY, MAXX, MAXY],
)
print(f'Clipped raster saved to: {result}')
info = geoai.get_raster_info(result)
for k, v in info.items():
    print(f'{k}: {v}')
"
```

Default output: `./clipped_<original_name>.tif`

### Stack bands

```bash
python3 -c "
import geoai

result = geoai.stack_bands(
    input_files=['FILE1', 'FILE2', 'FILE3'],
    output_file='OUTPUT_PATH',
)
print(f'Stacked raster saved to: {result}')
info = geoai.get_raster_info(result)
for k, v in info.items():
    print(f'{k}: {v}')
"
```

### Mosaic GeoTIFFs

```bash
python3 -c "
import geoai

result = geoai.mosaic_geotiffs(
    input_dir='INPUT_DIR',
    output_file='OUTPUT_PATH',
)
print(f'Mosaic saved to: {result}')
info = geoai.get_raster_info(result)
for k, v in info.items():
    print(f'{k}: {v}')
"
```

### Raster to vector

```bash
python3 -c "
import geoai

gdf = geoai.raster_to_vector(
    raster_path='INPUT_PATH',
    output_path='OUTPUT_PATH',
)
print(f'Vectorized: {len(gdf)} features')
print(f'Saved to: OUTPUT_PATH')
print(f'Columns: {list(gdf.columns)}')
"
```

Default output: `./<original_name>.gpkg`

### Vector to raster

```bash
python3 -c "
import geoai

result = geoai.vector_to_raster(
    vector_path='INPUT_PATH',
    output_path='OUTPUT_PATH',
    pixel_size=PIXEL_SIZE,
)
print(f'Rasterized: {result}')
info = geoai.get_raster_info(result)
for k, v in info.items():
    print(f'{k}: {v}')
"
```

Default pixel size: 1.0 (or infer from context). Default output: `./<original_name>.tif`

Replace all placeholder values with actual paths and parameters before running.

## Step 4 -- Update state

If a state directory exists, update it with the output file path using the same state resolution pattern as Step 2.

## Step 5 -- Report and suggest

Report:
- Operation performed
- Input and output file paths
- Key properties of the output (dimensions, CRS, band count, feature count)

Then suggest: *"Use `/geoai-skills:inspect-geo` to examine the result in detail."*

## Error handling

- **`import geoai` fails** -> delegate to `/geoai-skills:install-geoai`.
- **File not found** -> use `find` to locate, suggest corrected path.
- **CRS mismatch** (for stack/mosaic) -> report the issue and suggest reprojecting first.
- **Insufficient disk space** -> report the error.
- **Memory error** (very large rasters) -> suggest processing in tiles or using a smaller extent.
