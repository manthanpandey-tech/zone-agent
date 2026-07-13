# Zone Agent

A self-contained, notebook-based agent that processes engineering-drawing / floor-plan
images, uses **classical computer vision** (OpenCV) to detect wall-bounded regions,
splits the drawing into separate cropped zones with drawn boundaries, and saves every
execution into a **timestamped run folder** so history is never overwritten.

Detection is deterministic and **runs fully offline** — no ML model, no weight download,
no GPU. Each image is processed in milliseconds.

All the code lives in **[`agent.ipynb`](agent.ipynb)** — there is no separate module to
import. Open the notebook and run it top to bottom.

## What it produces (per run)

- The original image with detected **zone boundaries** drawn on it.
- One **cropped image per zone**.
- **Metadata per zone**: `zone_id`, `bbox` (`[x1, y1, x2, y2]`), `confidence`, `label`,
  `area`, and `file_path`.

## Folder layout

```
zone-agent/
  agent.ipynb            # the agent — all code, inline in cells
  requirements.txt       # dependencies
  README.md              # this file
  images/                # drop your input images here
  runs/                  # created at runtime — one timestamped folder per execution
    <YYYYMMDD-HHMMSS>/
      input/             # copy of the images used in this run
      output/
        <img>_annotated.png     # original with boundaries drawn
        <img>_metadata.json     # per-image zone metadata
        zones/<img>_zone_NN.png # cropped zones
      run/
        config.json      # exact config used (reproducibility)
        run.log          # execution log
        summary.json     # aggregate run summary
```

## Setup

Requires Python 3.9+. No CUDA, no internet, no model download.

```bash
cd zone-agent
python3 -m venv .venv
source .venv/bin/activate                 # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python -m ipykernel install --user --name zone-agent --display-name "Python (zone-agent)"
```

## Usage

1. Drop one or more floor-plan images (`.png`, `.jpg`, …) into [`images/`](images/).
   A few sample floor plans are already included so you can run it immediately.
2. Open [`agent.ipynb`](agent.ipynb) and select the **Python (zone-agent)** kernel.
3. Run all cells. A new `runs/<timestamp>/` folder is created with all outputs.
4. To process a different image set, replace the files in `images/` (or set `SOURCE_DIR`
   in the notebook to another folder) and re-run — previous runs are preserved.

## Configuration

Edit the `Config` in the notebook's example-usage cell:

| Field | Purpose | Default |
|-------|---------|---------|
| `ink_threshold` | Pixels darker than this (0–255) are treated as ink (walls/text) | `245` |
| `wall_line_frac` | Min wall-line length, as a fraction of the image size | `0.04` |
| `door_close_frac` | Kernel size for sealing doorway gaps (fraction of shorter side) | `0.035` |
| `min_area_frac` | Ignore enclosed regions smaller than this fraction of the image | `0.006` |
| `max_area_frac` | Ignore regions larger than this fraction of the image | `0.55` |
| `max_extent_frac` | Drop boxes spanning ~the whole plan in both dimensions | `0.92` |
| `min_side_px` | Drop boxes thinner than this many pixels on either side | `10` |
| `runs_base` | Base folder for run history | `runs` |

**Tuning**
- Too few zones / rooms merged → raise `ink_threshold` (faint walls) and/or `door_close_frac`.
- Too many zones / slivers → raise `min_area_frac` and/or `wall_line_frac`.
- A giant box around the whole plan → lower `max_extent_frac` or `max_area_frac`.

## How it works

```
load_images ─▶ looks_like_floor_plan (advisory, logged only)
                   │
                   ▼
extract_walls ─▶ detect_zones ─▶ draw_boundaries ─▶ save_outputs
   (wall mask)        │                 │
                      │                 └─▶ crop_zones ─▶ metadata
                      └────────── orchestrated by run_agent() ──────────
```

**The algorithm.** Everything darker than `ink_threshold` is treated as ink. Long
horizontal/vertical runs are kept as **walls** (text and furniture, being short/blobby,
drop out), and doorway-sized gaps are closed so each room becomes fully enclosed. The wall
mask is inverted to get the **free space**; its connected components are the candidate
rooms. The exterior (anything touching the image border), slivers, and whole-plan blobs
are filtered out, and each surviving region becomes one numbered zone (ordered
top-to-bottom, left-to-right).

Because it is pure geometry, results are deterministic — the same drawing always yields
the same zones — so `confidence` is always `1.0` (kept only for schema compatibility).

## Limitations

Room segmentation from arbitrary drawings is a heuristic, not a solved problem. Expect to
tune the `Config` per drawing style. Known failure modes:

- **Over-splitting** — features inside a room (a staircase, a closet, thick furniture
  linework) can carve it into several sub-zones. Raise `min_area_frac` / `wall_line_frac`.
- **Merging** — open-plan spaces or wide doorways with no wall between them collapse into a
  single zone. Raise `door_close_frac` to seal wider gaps, or `ink_threshold` for faint walls.
- **Spurious frame zones** — an outer dimension frame can register as a zone. Lower
  `max_extent_frac` / `max_area_frac` to reject it.
- **Non-rectilinear plans** — the wall extractor keeps horizontal/vertical runs, so heavily
  angled or curved walls are detected less reliably.

Zones are axis-aligned bounding boxes of enclosed regions — not pixel-exact room masks. For
tighter, label-aware zoning (one zone per *named* room), OCR-anchored seeding or a
distance-transform watershed would be the next step up.
