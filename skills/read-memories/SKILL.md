---
name: read-memories
description: >
  Search past Claude Code session logs to recover context from previous
  conversations. Finds past decisions, data paths, CRS info, model
  configurations, and unresolved work. Works across all projects or
  scoped to the current one.
argument-hint: <keyword> [--here] [--geo]
allowed-tools: Bash, Read
---

Use this to recover your own context from past sessions. Do NOT narrate the
process to the user -- silently run the query, absorb the results, and continue
with enriched context.

`$0` is the keyword to search for.
Pass `--here` as an argument to scope the search to the current project only.
Pass `--geo` as an argument to additionally extract geospatial-specific context
(EPSG codes, bounding boxes, CRS info, spatial file paths, model names).

## Step 1 -- Set the search path

```bash
ALL_PROJECTS="$HOME/.claude/projects/*/*.jsonl"
CURRENT_PROJECT="$HOME/.claude/projects/$(echo "$PWD" | sed 's|[/_]|-|g')/*.jsonl"
```

Use `$CURRENT_PROJECT` if any argument is `--here`, otherwise use `$ALL_PROJECTS`.
Store the chosen glob in `SEARCH_PATH`.

Check whether the `--geo` flag is present.

## Step 2 -- Query with Python

Run the following Python script via `python3 -c "..."`, substituting
`<SEARCH_PATH>` and `<KEYWORD>` with the resolved values. Escape any
single quotes in `<KEYWORD>` before embedding it.

```bash
python3 -c "
import json, glob, os

SEARCH_PATH = '<SEARCH_PATH>'
KEYWORD = '<KEYWORD>'.lower()
LIMIT = 40

files = sorted(glob.glob(os.path.expanduser(SEARCH_PATH)))
results = []

for fpath in files:
    parts = fpath.split('/')
    try:
        proj_idx = parts.index('projects') + 1
        project = parts[proj_idx] if proj_idx < len(parts) else 'unknown'
    except ValueError:
        project = 'unknown'

    with open(fpath, 'r', errors='replace') as f:
        for line in f:
            try:
                obj = json.loads(line)
            except (json.JSONDecodeError, ValueError):
                continue

            msg = obj.get('message')
            if not isinstance(msg, dict):
                continue
            role = msg.get('role')
            if role not in ('user', 'assistant'):
                continue

            content = msg.get('content', '')
            if isinstance(content, list):
                text = ' '.join(
                    c.get('text', '')
                    for c in content
                    if isinstance(c, dict) and 'text' in c
                )
            elif isinstance(content, str):
                text = content
            else:
                continue

            if KEYWORD not in text.lower():
                continue

            ts = obj.get('timestamp', '')
            snippet = text[:1500]
            results.append({
                'project': project,
                'ts': ts[:16].replace('T', ' ') if ts else '',
                'role': role,
                'content': snippet,
            })

            if len(results) >= LIMIT:
                break
    if len(results) >= LIMIT:
        break

print(f'Found {len(results)} results (limit {LIMIT})')
print('---')
for i, r in enumerate(results):
    print(f'[{i+1}] project={r[\"project\"]} ts={r[\"ts\"]} role={r[\"role\"]}')
    print(r['content'][:800])
    print('---')
"
```

## Step 3 -- Handle large result sets

If Step 2 reports exactly 40 results (limit hit), the keyword is common. Run a
counting pass to understand the scope:

```bash
python3 -c "
import json, glob, os

SEARCH_PATH = '<SEARCH_PATH>'
KEYWORD = '<KEYWORD>'.lower()

files = sorted(glob.glob(os.path.expanduser(SEARCH_PATH)))
total = 0
by_project = {}

for fpath in files:
    parts = fpath.split('/')
    try:
        proj_idx = parts.index('projects') + 1
        project = parts[proj_idx] if proj_idx < len(parts) else 'unknown'
    except ValueError:
        project = 'unknown'

    with open(fpath, 'r', errors='replace') as f:
        for line in f:
            try:
                obj = json.loads(line)
            except (json.JSONDecodeError, ValueError):
                continue
            msg = obj.get('message')
            if not isinstance(msg, dict):
                continue
            role = msg.get('role')
            if role not in ('user', 'assistant'):
                continue
            content = msg.get('content', '')
            if isinstance(content, list):
                text = ' '.join(
                    c.get('text', '')
                    for c in content
                    if isinstance(c, dict) and 'text' in c
                )
            elif isinstance(content, str):
                text = content
            else:
                continue
            if KEYWORD in text.lower():
                total += 1
                by_project[project] = by_project.get(project, 0) + 1

print(f'Total matches: {total}')
for proj, cnt in sorted(by_project.items(), key=lambda x: -x[1]):
    print(f'  {proj}: {cnt}')
"
```

Use this breakdown to decide whether to:
- Narrow the keyword (combine with a second term)
- Scope to `--here` if not already scoped
- Retrieve only the most recent results (sort by timestamp descending)

## Step 4 -- Extract geospatial context (when --geo is set)

If the `--geo` flag was provided, run an additional extraction pass:

```bash
python3 -c "
import json, glob, os, re

SEARCH_PATH = '<SEARCH_PATH>'
KEYWORD = '<KEYWORD>'.lower()

patterns = {
    'epsg_codes': re.compile(r'EPSG[:\s]*(\d{4,5})', re.IGNORECASE),
    'bbox': re.compile(r'(?:bbox|bounding.?box|bounds)\s*[=:]\s*\[([^\]]+)\]', re.IGNORECASE),
    'crs': re.compile(r'(?:CRS|SRS|projection)\s*[=:]\s*[\"\\']?([^\"\\'\\n,;]{3,60})', re.IGNORECASE),
    'spatial_files': re.compile(r'[\w/.-]+\.(?:shp|gpkg|geojson|tiff?|nc|hdf[45]?|gdb|fgb|kml|las|laz|parquet)', re.IGNORECASE),
    'coords': re.compile(r'(?:lat(?:itude)?|lon(?:gitude)?|lng)\s*[=:]\s*(-?\d+\.?\d*)', re.IGNORECASE),
    'models': re.compile(r'(?:sam2?|segment.?anything|yolo\w*|resnet\w*|u-?net|deeplabv3|mask.?rcnn|faster.?rcnn|swin|vit|dinov?\d?|geoclip|satlas|clay|prithvi)', re.IGNORECASE),
    'resolutions': re.compile(r'(\d+(?:\.\d+)?)\s*(?:m|meter|cm|km)\s*(?:resolution|pixel|spacing)', re.IGNORECASE),
}

files = sorted(glob.glob(os.path.expanduser(SEARCH_PATH)))
findings = {k: set() for k in patterns}

for fpath in files:
    with open(fpath, 'r', errors='replace') as f:
        for line in f:
            try:
                obj = json.loads(line)
            except (json.JSONDecodeError, ValueError):
                continue
            msg = obj.get('message')
            if not isinstance(msg, dict):
                continue
            role = msg.get('role')
            if role not in ('user', 'assistant'):
                continue
            content = msg.get('content', '')
            if isinstance(content, list):
                text = ' '.join(
                    c.get('text', '')
                    for c in content
                    if isinstance(c, dict) and 'text' in c
                )
            elif isinstance(content, str):
                text = content
            else:
                continue
            if KEYWORD not in text.lower():
                continue

            for name, pat in patterns.items():
                for m in pat.finditer(text):
                    findings[name].add(m.group(0).strip())

print('=== Geospatial Context ===')
for name, vals in findings.items():
    if vals:
        print(f'{name}:')
        for v in sorted(vals)[:20]:
            print(f'  - {v}')
"
```

## Step 5 -- Internalize

From the results, extract:
- Decisions made and their rationale
- Patterns and conventions established (coordinate systems, data formats, naming)
- Data file paths and datasets previously used
- CRS/EPSG codes that were chosen and why
- Bounding boxes or areas of interest
- Model configurations (architecture, hyperparameters, checkpoints)
- Unresolved items or open TODOs
- Any corrections the user made to your prior behavior

Use this to inform your current response. Do not repeat back the raw logs
to the user.

## Notes

- **No external dependencies**: This skill uses only Python standard library
  modules (json, glob, os, re). No pip install is needed.
- **Privacy**: All data stays local. Nothing is sent over the network.
- **Content types**: The search covers both user messages and assistant
  responses. It skips system messages, tool_use blocks, and tool_result
  blocks (only the `text` type within content arrays is extracted).
