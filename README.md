# Lithology Segmentation and Clustering

Extract depth-aligned lithology intervals from scanned mud logs, then explore them in **Core Explorer**, a local browser application for viewing logs, comparing wells, and finding similar rock patterns.

The Python pipeline combines OCR, image segmentation, texture clustering, and legend matching. It takes a scan and three annotated regions—the depth column, interpreted lithology column, and lithology legend—and produces interval tables and visual reports. It uses each page's symbols without requiring a trained lithology classifier or model weights.

## Features

- **Depth calibration:** reads depth labels with Tesseract, rejects inconsistent readings, and maps image rows to depth.
- **Interval extraction:** detects changes in texture and colour, groups similar intervals, and matches them to the page's legend.
- **Reviewable predictions:** preserves confidence scores, ambiguous matches, alternative names, and calibration warnings.
- **Visual reports:** exports CSV, detailed JSON, overview overlays, and close-ups for tall scans.
- **Interactive exploration:** provides a log library, scan viewer, side-by-side correlation, 2D/3D well maps, an interval similarity view, and field summaries.
- **New-log scanning:** accepts image or PDF uploads in the GUI, with editable or manually drawn bounding boxes.

## Quick start

Run these commands from the repository root. Use Python 3.10 or newer and a modern browser. There is no JavaScript build step; the frontend libraries are bundled in `gui/vendor/`.

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r gui/requirements.txt
```

On Windows, activate the environment with `.venv\Scripts\activate` instead.

Install **Tesseract OCR with English language data** separately from the Python packages:

```bash
# macOS (Homebrew)
brew install tesseract

# Debian / Ubuntu
sudo apt install tesseract-ocr tesseract-ocr-eng
```

On other platforms, install Tesseract through the appropriate installer or package manager and ensure `tesseract` is on your `PATH`. Check it with `tesseract --version`.

Start Core Explorer:

```bash
python gui/server.py
```

Open **http://localhost:8765** if the browser does not open automatically. The server binds to `127.0.0.1`; stop it with `Ctrl+C`. To change the port or suppress browser opening:

```bash
python gui/server.py --port 8766 --no-browser
```

The prebuilt library in `gui/data/` contains **48 logs grouped into 28 wells**. OCR and extraction run locally. Map backgrounds request external map tiles and require an internet connection.

### Scan a new log

1. Open **Scan a log** and upload a TIFF, PNG, JPEG, or PDF. For a supplied example, select both `validation_dataset/images/Nichols 1 Mud Log.tif` and `validation_dataset/annotations/Nichols 1 Mud Log.json`.
2. Check the three regions on the preview. Load a bounding-box JSON, adjust its boxes, or draw the boxes manually.
3. Select **Scan this log**. Review the intervals, confidence, legend, and CSV download under **Scanned logs**, or open the log report and 3D view.

Image uploads retain their original pixels and DPI. PDF pages are rendered, and supplied boxes are scaled to the rendered page. Uploaded files and pipeline outputs are stored under `gui/uploads/`; `gui/data/uploads.js` indexes completed scans.

Well positions may be inferred from a library match or a lease block in the filename. These positions are approximate unless verified; a location can also be entered manually.

## Command-line extraction

For extraction without the GUI, install `litho/requirements.txt` instead of `gui/requirements.txt`. Both workflows require the Tesseract binary. The GUI requirements already include the extraction dependencies and add PDF support through PyMuPDF.

Process one image and its annotation:

```bash
python litho/run.py \
  "validation_dataset/images/Nichols 1 Mud Log.tif" \
  "validation_dataset/annotations/Nichols 1 Mud Log.json" \
  --out results/
```

Process an entire dataset:

```bash
python litho/run.py --dir dataset/ --out results/dataset/
python litho/run.py --dir validation_dataset/ --out results/validation/
```

`--dir` searches recursively and pairs images with annotations using `image_name`, then matching stems. The CLI accepts `.tif`, `.tiff`, `.png`, `.jpg`, and `.jpeg`; use the GUI for PDFs. Add `--limit 1` for a small trial run.

The batch runner reports warnings and failures and continues with the remaining files. Check its final `ok` / `failed` summary: a completed command does not guarantee every image was processed.

### Annotation format

Each annotation describes the original image dimensions and three bounding boxes:

```json
{
  "image_name": "example.tif",
  "width": 1600,
  "height": 12000,
  "bounding_boxes": {
    "depth_column": [200, 1500, 300, 11500],
    "interpreted_lithology_column": [900, 1500, 1000, 11500],
    "lithology_legend": [20, 500, 1500, 1100]
  }
}
```

Boxes use `[x_min, y_min, x_max, y_max]` in full-image pixels, with the origin at the top left and exclusive maximum coordinates. The example illustrates the schema; use boxes measured on your own image. The loader rejects mismatched image dimensions or boxes outside the image.

Depth calculations and segmentation thresholds assume **feet**. MD and TVD follow the depth track selected by the annotation; the pipeline does not convert between them.

### Output files

For an input named `example.tif`, the output directory contains:

| File | Contents |
| --- | --- |
| `example.csv` | One row per extracted lithology interval. |
| `example.json` | Intervals, confidence, ambiguity, alternatives, depth calibration, parsed legend, snapping information, warnings, and timings. |
| `example_overlay.png` | Overview of the scan's depth and lithology tracks alongside extracted intervals. |
| `example_detail_1.png`, etc. | Full-resolution close-ups when the log is too tall for the overview. |

The CSV has exactly these columns:

```text
top_depth,bottom_depth,lithology,cluster_id,top_y,bottom_y
```

Depths are in feet. `top_y` and `bottom_y` are full-image pixel rows; `cluster_id` identifies a group within that log, not a shared lithology ID across wells. Before writing outputs, the pipeline checks that intervals have positive thickness, remain within the lithology column, and meet without gaps or overlaps in both depth and pixels. Blank stretches at the start and end are excluded from the logged range.

## How the pipeline works

| Stage | Implementation | Approach |
| --- | --- | --- |
| Load regions | `io.py` | Keeps the scan in its native image mode and converts cropped regions to RGB to limit memory use. |
| Calibrate depth | `depth.py` | OCRs depth labels, determines orientation, rejects outliers, and interpolates between accepted labels. Tall logs use sampled chunks where possible. |
| Segment and cluster | `features.py`, `segment.py` | Measures row texture, ink, colour, and stroke features; detects boundaries with a window informed by symbol repetition; clusters interval descriptors. |
| Read the legend | `legend.py` | Detects swatches and nearby rock names, cleans OCR text, and recognises boxed and some unboxed symbols. |
| Assign lithology | `shape.py`, `tiles.py`, `pipeline.py` | Matches stroke and fill descriptors, or directly matches repeated legend drawings where suitable. Retains alternatives for ambiguous symbols. |
| Refine intervals | `snap.py`, `pipeline.py` | Merges adjacent equal labels and snaps boundaries when a regular depth grid is detected. |
| Render reports | `overlay.py` | Draws overview and detail images for inspection against the source scan. |

All implementation paths in this table are under `litho/lithoextract/`.

## Included data and results

| Collection | Annotated scans | Saved result sets | Extracted intervals |
| --- | ---: | ---: | ---: |
| `dataset/` → `dataset_results/` | 51 | 48 | 11,431 |
| `validation_dataset/` → `validation_results/` | 3 | 3 | 628 |

These counts describe the artifacts included in this repository, not a fresh benchmark or a measure of prediction accuracy. Three primary-dataset scans have no saved result set. The annotations mark input regions; they do not provide interval-level lithology ground truth for accuracy scoring. Saved outputs may differ from a rerun with the current pipeline or another OCR version.

### Rebuild the GUI library

The GUI bundle is generated from extraction results and the original scans. To rebuild it from the included results, specify `dataset_results/` explicitly because the script's default is `results/`:

```bash
python gui/build_data.py \
  --results dataset_results/ \
  --images dataset/images/ \
  --bboxes dataset/bbox_annotation/
```

This writes `gui/data/data.js`, scan tiles, legend thumbnails, and a rebuild cache. If you generated new results, set `--results` to their directory. Use `--no-tiles` to skip tile generation when the required tiles already exist.

## Repository layout

```text
.
├── litho/                    # Python extraction CLI and lithoextract package
├── gui/                      # Core Explorer server, frontend, and bundled libraries
│   └── data/                 # Prebuilt library, scan tiles, and legend thumbnails
├── dataset/                  # Primary scans, bounding-box annotations, and previews
├── dataset_results/          # Saved primary-dataset CSV, JSON, and image reports
├── validation_dataset/       # Three additional scans and their annotations
├── validation_results/       # Saved outputs for the additional scans
├── example_notebook.ipynb    # Loading, annotation, and result-visualisation walkthrough
├── brief.pptx               # Project brief
└── presentation.pptx        # Project presentation
```

The notebook retains paths to an earlier `example/` directory. To reuse it, point its image, annotation, and CSV variables at matching files in `dataset/` and `dataset_results/`; treat annotation metadata such as `coordinate_system` as optional when reading it.

## Limitations

- Depth OCR can fail on faint or unreadable labels. Fits based on few labels or inconsistent scales need inspection; review the JSON diagnostics and the scan together.
- Beds thinner than a repeated symbol on compressed logs can be approximate or missed. Fading, mixed patterns, and scan artifacts also affect boundaries.
- Identical legend symbols cannot be distinguished visually. The pipeline selects a name using a rock-frequency heuristic and marks alternatives as ambiguous.
- Legend OCR can misread names or omit part of a multiline label. Insufficient usable legend symbols produce unidentified clusters.
- Confidence scores reflect matching heuristics, not calibrated probabilities of correctness.
- Well grouping and filename-derived map locations are heuristic. Logs on MD and TVD tracks need compatible depth references before geological comparison.

The repository includes third-party frontend libraries and their license notices in `gui/vendor/`.
