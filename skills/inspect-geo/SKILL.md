---
name: inspect-geo
description: >
  Inspect any raster or vector geospatial file. Returns CRS, bounds, bands,
  resolution, dtype, attribute summaries, and band statistics. Supports
  GeoTIFF, Shapefile, GeoJSON, GeoPackage, GeoParquet, and more.
argument-hint: <filepath> [question about the data]
allowed-tools: Bash
---

You are helping the user inspect a geospatial data file.

Filename given: `$0`
Question: `${1:-describe the data}`

Follow these steps in order, stopping and reporting clearly if any step fails.

## Step 1 -- Resolve the file path

If `$0` looks like an absolute path, use it directly. Otherwise search for it:

```bash
find "$PWD" -name "$0" -not -path '*/.git/*' 2>/dev/null
```

- **Zero results** -> tell the user the file was not found and stop.
- **More than one result** -> list all matches, ask the user to re-run with a fuller path, and stop.
- **Exactly one result** -> use that full path as `RESOLVED_PATH`.

## Step 2 -- Classify the file type

Determine whether the file is raster or vector based on its extension:

- **Raster**: `.tif`, `.tiff`, `.img`, `.jp2`, `.vrt`, `.nc`, `.hdf`
- **Vector**: `.geojson`, `.json`, `.shp`, `.gpkg`, `.parquet`, `.geoparquet`, `.fgb`, `.kml`

If the extension is ambiguous, try raster first, then vector.

## Step 3 -- Run the appropriate inspection

### Raster files

```bash
python3 -c "
import geoai

info = geoai.get_raster_info('RESOLVED_PATH')
for k, v in info.items():
    print(f'{k}: {v}')

print('---')
print('Band Statistics:')
stats = geoai.get_raster_stats('RESOLVED_PATH')
for k, v in stats.items():
    print(f'{k}: {v}')
"
```

### Vector files

```bash
python3 -c "
import geoai

info = geoai.get_vector_info('RESOLVED_PATH')
for k, v in info.items():
    print(f'{k}: {v}')
"
```

Replace `RESOLVED_PATH` with the actual absolute path before running.

## Step 4 -- Answer the user's question

Using the metadata retrieved in Step 3, answer:

`${1:-describe the data: summarize the file type, CRS, extent, and any notable properties.}`

For vector files, if the question references a specific attribute, run an additional analysis:

```bash
python3 -c "
import geoai
result = geoai.analyze_vector_attributes('RESOLVED_PATH')
print(result)
"
```

## Step 5 -- Update state

Resolve the state directory:

```bash
STATE_DIR=""
test -f .geoai-skills/state.json && STATE_DIR=".geoai-skills"
PROJECT_ROOT="$(git rev-parse --show-toplevel 2>/dev/null || echo "$PWD")"
PROJECT_ID="$(echo "$PROJECT_ROOT" | tr '/' '-')"
test -f "$HOME/.geoai-skills/$PROJECT_ID/state.json" && STATE_DIR="$HOME/.geoai-skills/$PROJECT_ID"
```

If `STATE_DIR` is set, update it with the inspected file info:

```bash
python3 -c "
import json, os
state_file = 'STATE_DIR/state.json'
state = {}
if os.path.exists(state_file):
    with open(state_file) as f:
        state = json.load(f)
state['last_inspected'] = {
    'path': 'RESOLVED_PATH',
    'type': 'TYPE',
}
with open(state_file, 'w') as f:
    json.dump(state, f, indent=2)
"
```

Replace `STATE_DIR`, `RESOLVED_PATH`, and `TYPE` (raster or vector) with actual values.

If no state directory exists yet, skip this step silently.

## Step 6 -- Suggest next steps

After reporting, briefly mention:

- For raster files: *"To process this raster (clip, mosaic, stack bands), use `/geoai-skills:process-raster`. To run AI detection on it, use `/geoai-skills:detect-objects`."*
- For vector files: *"To download more vector data for this area, use `/geoai-skills:overture-data`."*

Keep suggestions brief and show them only once.

## Error handling

- If `python3` is not found or `import geoai` fails, delegate to `/geoai-skills:install-geoai`.
- If rasterio or geopandas fails to read the file, try the GDAL/OGR fallback:

```bash
python3 -c "
import geoai
info = geoai.get_raster_info_gdal('RESOLVED_PATH')
for k, v in info.items():
    print(f'{k}: {v}')
"
```

Or for vectors:

```bash
python3 -c "
import geoai
info = geoai.get_vector_info_ogr('RESOLVED_PATH')
for k, v in info.items():
    print(f'{k}: {v}')
"
```
