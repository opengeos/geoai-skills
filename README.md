# geoai-skills

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugin that adds GeoAI-powered skills for geospatial data exploration, satellite imagery download, AI-based object detection, and session memory.

Built on the [GeoAI](https://opengeoai.org) Python library.

## Installation

### From GitHub

Add the repository as a plugin source and install:

```
/plugin marketplace add opengeos/geoai-skills
/plugin install geoai-skills@geoai-skills
```

This registers the GitHub repo as a marketplace and installs the plugin. Skills will be available as `/geoai-skills:<skill-name>` in all future sessions.

### Updating

To pull the latest version, update the marketplace first and then the plugin:

```
/plugin marketplace update geoai-skills
/plugin update geoai-skills@geoai-skills
```

## Prerequisites

- Python 3.10+ with the `geoai-py` package installed (`pip install geoai-py`)
- For AI models (object detection, segmentation): PyTorch with CUDA (optional, CPU works but is slower)
- For Overture Maps data: `pip install "geoai-py[extra]"`

## Skills

### `inspect-geo`

Inspect any raster or vector geospatial file -- CRS, bounds, bands, resolution, dtype, attribute summaries, and band statistics. Supports GeoTIFF, Shapefile, GeoJSON, GeoPackage, GeoParquet, and more.

```
/geoai-skills:inspect-geo dem.tif
/geoai-skills:inspect-geo buildings.gpkg what CRS is this in?
/geoai-skills:inspect-geo landcover.tif how many bands does it have?
```

### `download-data`

Download NAIP aerial imagery for a bounding box. Specify coordinates as minx,miny,maxx,maxy in WGS84 and optionally a year.

```
/geoai-skills:download-data -83.5,35.5,-83.4,35.6 --year 2022
/geoai-skills:download-data -122.5,37.7,-122.3,37.8 --output ./naip/
```

### `search-stac`

Search and download satellite imagery from Microsoft Planetary Computer. Browse collections, search by bbox and time range, and download items.

```
/geoai-skills:search-stac list
/geoai-skills:search-stac sentinel-2-l2a --bbox -83.5,35.5,-83.4,35.6 --datetime 2023-01-01/2023-06-30
/geoai-skills:search-stac naip --bbox -83.5,35.5,-83.4,35.6 --download
```

### `overture-data`

Download Overture Maps data (buildings, places, roads, land use, water, etc.) for a bounding box.

```
/geoai-skills:overture-data building --bbox -83.5,35.5,-83.4,35.6
/geoai-skills:overture-data land_use --bbox -122.5,37.7,-122.3,37.8 --output land_use.gpkg
```

### `process-raster`

Process raster data: clip by bounding box, stack multiple bands, mosaic GeoTIFFs, or convert between raster and vector formats.

```
/geoai-skills:process-raster clip input.tif --bbox -83.5,35.5,-83.4,35.6
/geoai-skills:process-raster mosaic ./tiles/ --output mosaic.tif
/geoai-skills:process-raster stack band1.tif band2.tif band3.tif --output stacked.tif
/geoai-skills:process-raster raster-to-vector classification.tif --output polygons.gpkg
```

### `detect-objects`

Run pre-trained AI models on geospatial imagery. Detect buildings, cars, ships, solar panels, or use text-prompted segmentation with GroundedSAM.

```
/geoai-skills:detect-objects buildings naip_image.tif
/geoai-skills:detect-objects cars parking_lot.tif --output car_detections.gpkg
/geoai-skills:detect-objects grounded-sam satellite.tif --text "swimming pools"
```

### `read-memories`

Search past Claude Code session logs to recover context from previous conversations -- past decisions, data paths, CRS info, model configurations, and unresolved work.

```
/geoai-skills:read-memories naip --here
/geoai-skills:read-memories building detection --geo
```

### `install-geoai`

Verify that the geoai Python package is installed and functional. Optionally check extra dependencies for deep learning models.

```
/geoai-skills:install-geoai
/geoai-skills:install-geoai --check
/geoai-skills:install-geoai --extras
```

## Session state

Skills can share state through a JSON file per project (`state.json`), containing information about recently inspected files, downloaded data paths, and working directories. When state is first needed, you will be asked where to store it:

1. **In the project directory** (`.geoai-skills/state.json`) -- colocated with the project, optionally gitignored
2. **In your home directory** (`~/.geoai-skills/<project-id>/state.json`) -- keeps the project directory clean

State is used to auto-fill file paths across skills. For example, after downloading NAIP imagery, `inspect-geo` and `detect-objects` can reference the downloaded files without re-specifying paths.

## How the skills work together

Skills reference each other where it makes sense:

- `inspect-geo` suggests `process-raster` for processing and `detect-objects` for AI analysis
- `download-data`, `search-stac`, and `overture-data` suggest `inspect-geo` for examining results
- `detect-objects` suggests `inspect-geo` for examining detection output
- All skills delegate to `install-geoai` when the geoai package is not available
- `read-memories` with `--geo` extracts geospatial-specific context from past sessions

A typical workflow:

```
/geoai-skills:download-data -83.5,35.5,-83.4,35.6 --year 2022
/geoai-skills:inspect-geo naip_image.tif
/geoai-skills:detect-objects buildings naip_image.tif
/geoai-skills:inspect-geo buildings_detections.gpkg
```

## Local development

To test skills locally from a clone of this repo:

```bash
# 1. Clone the repo
git clone https://github.com/opengeos/geoai-skills.git
cd geoai-skills

# 2. Launch Claude Code with the local plugin directory
claude --plugin-dir .
```

This loads the plugin from disk instead of the marketplace, so any edits to `skills/*/SKILL.md` take effect immediately -- just start a new conversation (or re-run the slash command) to pick up changes.

## Platform support

These skills have been tested on **macOS** and **Linux**. Windows is not yet fully supported -- some shell commands and path handling may not work as expected.

## Reporting issues

Found a bug or have an idea for improvement? Open an issue at:

**https://github.com/opengeos/geoai-skills/issues**

For geoai-specific bugs (model loading, data download errors), please include the geoai version (`python -c "import geoai; print(geoai.__version__)"`) and the full error message.
