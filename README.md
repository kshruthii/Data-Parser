# ParseAnything

**A universal document ingestion engine.** Upload a PDF, DOCX, PPTX, XLSX or an image (PNG, JPG) and get back a single canonical structure: ordered blocks (headings, paragraphs, lists, tables, figures, charts, equations), each with a confidence score, a review flag and exact provenance. Output is delivered as JSON and Markdown.

Built for the DataQuest 3.0 hackathon. FastAPI backend, React frontend.

---

## Contents

- [Highlights](#highlights)
- [Supported formats](#supported-formats)
- [Languages](#languages)
- [How it works](#how-it-works)
- [Quick start](#quick-start)
- [Configuration](#configuration)
- [Command reference](#command-reference)
- [REST API](#rest-api)
- [Output format](#output-format)
- [Confidence and review](#confidence-and-review)
- [Search and summary](#search-and-summary)
- [Frontend](#frontend)
- [Optional engines](#optional-engines)
- [Testing](#testing)
- [Project structure](#project-structure)
- [Known limitations](#known-limitations)
- [Troubleshooting](#troubleshooting)

---

## Highlights

| Capability | Detail |
|---|---|
| **One schema for every format** | PDF, DOCX, PPTX, XLSX and images all produce the same `Document` / `Block` model. |
| **Digital and scanned input** | Per-page detection; scanned PDF pages and image files go through OCR (Tesseract, or PaddleOCR if installed). Mixed documents work. |
| **Tables** | Ruled and borderless detection, merged cells (row/col span), multi-row headers, cross-page table merging. |
| **Charts** | Vector bar, line, pie, scatter and histogram charts converted to structured values via axis calibration. |
| **Equations** | Geometry-based LaTeX reconstruction (scripts, fractions, radicals, sums, integrals, Greek). |
| **Multilingual (English, Hindi, Tamil)** | OCR runs with `eng+hin+tam` by default. Missing language packs degrade gracefully to what is installed. |
| **Reading order** | Multi-column and sidebar aware; repeated headers, footers and page numbers are stripped from Markdown. |
| **Provenance** | Every block records document, page, bounding box, extractor and all merged source regions. |
| **Honest confidence** | Scores come from real extraction signals. Anything below threshold is marked `REVIEW_REQUIRED`. |
| **Fail-safe** | A failing region never aborts the document. Result is `SUCCESS`, `PARTIAL_SUCCESS` or `FAILED` with structured error codes. |
| **Search and summary** | Phrase search with bounding-box results; deterministic, extractive, LLM-free document summary. |
| **Measured performance** | Real `perf_counter` timings: total, per page, pages per second, per pipeline stage. |

---

## Supported formats

| Format | Location model | Source viewer in UI |
|---|---|---|
| **PDF** (primary) | Physical pages, bounding boxes in PDF points | Page image with bbox highlight |
| **DOCX** | One logical unit; paragraph / table / image indices. No bbox is invented. | Structural anchors |
| **PPTX** | One logical page per slide; shape bboxes converted from EMU to points | Slide preview with element highlight |
| **XLSX** | One logical page per worksheet; sheet, range and cell addresses | Worksheet grid with range highlight |
| **Image** (`.png`, `.jpg`, `.jpeg`) | Treated as a scanned page and read with OCR (English, Hindi, Tamil) | Image preview |

Images embedded inside DOCX and PPTX files are extracted as figures, and PPTX pictures are also OCR'd (English, Hindi, Tamil). Unsupported extensions return a structured `UNSUPPORTED_FORMAT` error.

---

## Languages

| Area | Behaviour |
|---|---|
| **Scanned PDF and image OCR** | Tesseract with `PARSE_OCR_LANGS` (default `eng+hin+tam`). Languages whose `traineddata` is not installed are skipped with a logged warning. |
| **PPTX embedded pictures** | OCR'd as `eng+hin+tam`, with small images upscaled to improve Devanagari accuracy. |
| **Digital text (PDF, DOCX, XLSX)** | Read directly from the file, so any Unicode script is preserved without OCR. |
| **Engine choice** | With `PARSE_OCR_ENGINE=auto`, Tesseract is used whenever a non-English language is available, since PaddleOCR is configured for English only here. |
| **Summary** | Extractive and English-oriented. Hindi and Tamil text is kept verbatim but sentence splitting and ranking are tuned for English. |

Install the language packs for your platform:

```bash
# Ubuntu / Debian
sudo apt-get install tesseract-ocr-hin tesseract-ocr-tam

# macOS
brew install tesseract-lang

# Windows: copy hin.traineddata and tam.traineddata into
#   C:\Program Files\Tesseract-OCR\tessdata

# Verify
tesseract --list-langs
```

To change languages, set `PARSE_OCR_LANGS`, for example `PARSE_OCR_LANGS=eng` for English only.

Hindi and Tamil OCR quality depends on the installed packs and on scan quality, so check accuracy on your own samples.

---

## How it works

```
Upload ──> Detect & route ──> Extract ──> Assemble ──> Confidence ──> Provenance ──> Summary ──> Output
              │                  │            │
              │                  │            └─ cross-page merge, reading order,
              │                  │               heading hierarchy, header/footer removal
              │                  └─ text · OCR · tables · figures · charts · equations
              └─ validate file, classify each page as digital or scanned
```

The canonical `Document` is the single source of truth. Markdown, search and summary are all derived from it; the source file is never re-parsed.

---

## Quick start

### Manual setup

**Prerequisites:** Python 3.11 or newer, Node.js 18 or newer, and the Tesseract binary for OCR.

**0. Get the code**

```bash
git clone https://github.com/kshruthii/Data-Parser.git
cd Data-Parser
```

**1. Install Tesseract**

```bash
# Ubuntu / Debian
sudo apt-get install tesseract-ocr tesseract-ocr-hin tesseract-ocr-tam

# macOS
brew install tesseract tesseract-lang

# Windows
winget install UB-Mannheim.TesseractOCR
```

**2. Backend**

```bash
cd backend
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt
uvicorn app.main:app --reload
```

**3. Frontend** (second terminal)

```bash
cd frontend
npm install
npm run dev
```

**4. Verify**

```bash
curl http://localhost:8000/health
# {"status":"ok"}
```

### Windows notes

```powershell
cd backend
py -3.12 -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
python -m pytest -q
uvicorn app.main:app --reload
```

Tesseract is auto-detected at `C:\Program Files\Tesseract-OCR\tesseract.exe`. Lookup order: `TESSERACT_CMD`, then `PATH`, then `Program Files`, then `Program Files (x86)`. Place extra language packs (`hin.traineddata`, `tam.traineddata`) in the `tessdata` folder.

Without Tesseract the API still runs, but scanned pages report an OCR error.

---

## Configuration

All settings are environment variables read at startup. Export them in your shell before starting the server. The `.env.example` file is a reference template and is **not loaded automatically**.

### Core

| Variable | Default | Description |
|---|---|---|
| `PARSE_BASE_DIR` | repository root | Base for `data/` and `models/` |
| `PARSE_UPLOAD_DIR` | `<base>/data/uploads` | Where uploads are stored |
| `PARSE_PROCESSED_DIR` | `<base>/data/processed` | Where results are written |
| `PARSE_MAX_UPLOAD_MB` | `50` | Maximum upload size |
| `CORS_ORIGINS` | `http://localhost:5173` | Comma-separated allowed origins |

### OCR

| Variable | Default | Description |
|---|---|---|
| `PARSE_OCR_ENGINE` | `auto` | `auto`, `paddle` or `tesseract` |
| `PARSE_OCR_LANGS` | `eng+hin+tam` | Tesseract language codes. Missing packs are skipped with a warning. |
| `PARSE_OCR_DPI` | `200` | Render resolution for scanned pages |
| `PARSE_OCR_DENOISE` | `0` | `1` enables OpenCV denoising (slow) |
| `PARSE_MIN_TEXT_CHARS` | `25` | A page with fewer text characters is treated as scanned |
| `TESSERACT_CMD` | auto-detected | Full path to the Tesseract executable |

### Confidence

| Variable | Default | Description |
|---|---|---|
| `PARSE_HIGH_THRESHOLD` | `0.85` | Score at or above this is `HIGH` |
| `PARSE_REVIEW_THRESHOLD` | `0.6` | Score below this is `LOW` and `REVIEW_REQUIRED` |

### Frontend

| Variable | Default | Description |
|---|---|---|
| `VITE_API_BASE` | `http://localhost:8000/api` | API base URL used by the UI |

---

## Command reference

There is no standalone CLI binary. The tool is driven through the REST API (shell examples below), a Python entry point, or the web UI.

### Server

```bash
cd backend && uvicorn app.main:app --reload
```

### Parse a file from the shell

```bash
# 1. Upload
curl -F "file=@report.pdf" http://localhost:8000/api/upload      # also .docx .pptx .xlsx .png .jpg
# {"file_id":"a1b2c3d4e5f6","document_id":"a1b2c3d4e5f6","filename":"report.pdf","size_bytes":59617}

# 2. Start parsing (asynchronous)
curl -X POST "http://localhost:8000/api/parse?file_id=a1b2c3d4e5f6"

#    (add &wait=true to block until finished)
# 3. Poll status
curl http://localhost:8000/api/status/a1b2c3d4e5f6

# 4. Fetch results
curl http://localhost:8000/api/result/a1b2c3d4e5f6                          # full JSON
curl -OJ http://localhost:8000/api/download/a1b2c3d4e5f6/json               # document.json
curl -OJ http://localhost:8000/api/download/a1b2c3d4e5f6/markdown           # document.md
```

### Use as a Python library

Run from the `backend` directory:

```python
from app.pipeline.orchestrator import parse_document

doc = parse_document("report.pdf")        # .pdf .docx .pptx .xlsx .png .jpg
print(doc.status, doc.page_count, doc.processing_time)

for block in doc.blocks:
    print(block.id, block.type.value, block.confidence_level.value, block.page)
```

Results are written to `data/processed/<document_id>/`.

### Tests

```bash
cd backend
python -m pytest -q
```

### Frontend

```bash
cd frontend
npm install
npm run dev        # http://localhost:5173
```

---

## REST API

Base path: `/api`. Interactive documentation is served at `/docs`.

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/health` | Liveness check (no `/api` prefix) |
| `POST` | `/api/upload` | Upload a file (multipart field `file`). Returns `file_id`. |
| `POST` | `/api/parse?file_id=<id>&wait=<bool>` | Start parsing. `wait=true` blocks until complete. |
| `GET` | `/api/status/{id}` | Job status, timing and top errors |
| `GET` | `/api/result/{id}` | Full canonical `document.json` |
| `GET` | `/api/download/{id}/{fmt}` | Download `json`, `markdown` or `md` |
| `POST` | `/api/search` | Search a parsed document |
| `GET` | `/api/documents/{id}/page/{n}.png` | Rendered PDF page (`dpi` 40 to 200, default 110). PDF only. |
| `GET` | `/api/documents/{id}/figures/{name}` | Extracted figure PNG |
| `POST` | `/api/export/markdown` | Render a posted document dict to Markdown |
| `POST` | `/api/export/json` | Serialise a posted document dict to JSON |

### Job lifecycle

`UPLOADED` → `QUEUED` → `PROCESSING` → `SUCCESS` | `PARTIAL_SUCCESS` | `FAILED`

Jobs run on an in-process thread pool (2 workers). Status is kept in memory and mirrored to `data/processed/<id>/status.json`, so it survives a restart.

### HTTP errors

| Code | When |
|---|---|
| `400` | Invalid document id, or empty upload |
| `404` | Unknown id, result not ready, page or figure not found |
| `413` | File exceeds `PARSE_MAX_UPLOAD_MB` |
| `422` | Blank search query, or page render failure |

### Document error codes

| Code | Severity | Meaning |
|---|---|---|
| `UNSUPPORTED_FORMAT` | error | Extension is not PDF, DOCX, PPTX or XLSX |
| `CORRUPT_PDF` / `CORRUPT_DOCX` | error | File cannot be opened |
| `ENCRYPTED_PDF` | error | Password-protected PDF |
| `FILE_TOO_LARGE` | error | Over the size limit |
| `OCR_FAILURE` | error | OCR failed for a page |
| `LAYOUT_DETECTION_FAILURE` | error | Region detection failed for a page |
| `TABLE_EXTRACTION_FAILURE` | error | A table could not be structured |
| `FORMULA_EXTRACTION_FAILURE` | error | Formula recognition failed |
| `PARSING_FAILED` | error | Unexpected pipeline failure |
| `LOW_CONFIDENCE` | warning | Block needs review |
| `READING_ORDER_AMBIGUITY` | warning | Page layout order is uncertain |

Final status: `FAILED` if there are errors and no blocks, `PARTIAL_SUCCESS` if there are errors alongside blocks, otherwise `SUCCESS`.

---

## Output format

Each parse writes three artefacts to `data/processed/<document_id>/`:

```
document.json     canonical structured output
document.md       Markdown rendered from document.json
figures/          extracted figure and chart crops (PNG)
status.json       job status
```

### Document

```json
{
  "document_id": "6e599290db44",
  "filename": "report.pdf",
  "format": "pdf",
  "page_count": 3,
  "processing_time": 0.256,
  "status": "SUCCESS",
  "pages":   [ ... ],
  "blocks":  [ ... ],
  "errors":  [ ... ],
  "stats":   { ... },
  "summary": { ... },
  "timing":  { ... }
}
```

### Block

```json
{
  "id": "B0004",
  "type": "paragraph",
  "content": "Revenue increased from $100 million in 2024 to $128 million in 2025.",
  "page": 1,
  "bbox": [57.02, 150.6, 523.34, 176.34],
  "confidence": 0.945,
  "confidence_level": "HIGH",
  "status": "OK",
  "extractor": "text-layer",
  "provenance": {
    "document": "6e599290db44",
    "page": 1,
    "bbox": [57.02, 150.6, 523.34, 176.34],
    "extractor": "text-layer",
    "source_region_id": "R1-6",
    "sources": [{ "page": 1, "bbox": [57.02, 150.6, 523.34, 176.34], "region_id": "R1-6" }]
  },
  "signals": { "...": 0.0 }
}
```

`bbox` is `[x0, y0, x1, y1]` in PDF points. It is `null` for DOCX and XLSX, where no physical coordinates exist.

### Block types

`heading` · `paragraph` · `list` · `table` · `figure` · `chart` · `equation` · `caption` · `header` · `footer` · `footnote` · `reference`

### Content by type

- **Chart:** `chart_type`, `title`, axes with ticks, and `series` of `{category, value}` points. Values come from axis calibration or printed data labels.
- **Equation:** `{raw_text, latex}`. `latex` is `null` when it cannot be reconstructed, and the block is flagged for review.
- **Table:** `headers`, `rows`, and `cells` with `row`, `col`, `rowspan`, `colspan`, `is_header`, plus `continued_from_previous` and `page_span` for tables merged across pages.

Markdown output renders equations as `$$` blocks and charts as an image plus a data table. Repeated headers, footers and page numbers are omitted from Markdown but kept in the JSON.

### Timing

`timing` reports `processing_time_seconds`, `pages_processed`, `seconds_per_page`, `pages_per_second` and per-stage seconds (`ingestion`, `ocr`, `layout_analysis`, `table_extraction`, `assembly`, `validation`, `output_generation`, `summary`). For DOCX, PPTX and XLSX, throughput is reported against `logical_units_processed` instead of fabricated physical pages.

---

## Confidence and review

Every block carries a `signals` dictionary of real extraction measurements (for example OCR word confidence, ruling-line coverage, geometry fit). The score is:

```
confidence = 0.6 × mean(signals) + 0.4 × min(signals)
```

| Level | Score | Status |
|---|---|---|
| `HIGH` | ≥ 0.85 | `OK` |
| `MEDIUM` | ≥ 0.60 | `OK` |
| `LOW` | < 0.60 | `REVIEW_REQUIRED` |

A block is also set to `REVIEW_REQUIRED` when the extractor gives an explicit `review_reason`, such as an equation without LaTeX, a chart without calibrated values, or a possible table continuation that was not merged. Each flagged block adds a `LOW_CONFIDENCE` warning to `errors`.

---

## Search and summary

### Search

`POST /api/search` runs against the saved `document.json`.

Request body: `{"document_id": "a1b2c3d4e5f6", "query": "revenue", "limit": 10}`

```json
{
  "document_id": "a1b2c3d4e5f6",
  "query": "revenue",
  "total_matches": 7,
  "returned": 2,
  "truncated": true,
  "results": [{
    "block_id": "B0004",
    "block_type": "paragraph",
    "page": 1,
    "bbox": [57.02, 150.6, 523.34, 176.34],
    "confidence": 0.945,
    "reading_order": 4,
    "matched_text": "Revenue",
    "snippet": "Revenue increased from $100 million in 2024 to $128 milli...",
    "match_count": 1
  }]
}
```

- Case-insensitive, partial and phrase matching. Whitespace in a query matches any whitespace run. Regex characters are treated literally.
- One hit per block, in reading order. `limit` ranges 1 to 1000 (default 200).
- Searches paragraph text, headings, captions, list items, table cells (each cell separately), equation text and LaTeX, chart titles, labels and categories, and OCR text.
- Not supported: fuzzy matching, stemming, or non-contiguous multi-word queries.

### Summary

Built into every result as `summary`. Fully local and deterministic: no network, no API key, no LLM.

- **Overview:** title, filename, pages, source kind (digital, scanned, mixed).
- **Executive summary:** a templated sentence of counted facts plus up to three extractive sentences copied verbatim, each traced to its `block_id`.
- **Key points:** up to six further sentences plus factual table and chart points.
- **Major sections:** top two heading levels.
- **Statistics:** pages, tables, figures, charts, equations, words, low-confidence and review-required counts.

Low-confidence blocks are excluded when reliable text exists. If everything is low confidence, the summary is returned with `status: "uncertain"` and a warning. `language` and `document_type` are always `null`, since neither is detected.

---

## Frontend

React 19 and Vite, with react-markdown and KaTeX.

- Upload, parse and live status polling
- **Summary** tab (default), **Markdown** (rendered or raw), **JSON** and **Blocks** views
- Confidence badges and review flags on every block
- Click a block or search hit to jump to its source: PDF page with bbox overlay, PPTX slide element, XLSX worksheet range, DOCX paragraph or table anchor
- Timing panel with per-stage breakdown
- JSON and Markdown download buttons

---

## Optional engines

Core install works with Tesseract alone. Extra engines are picked up automatically when importable.

```bash
# PaddleOCR (alternative OCR engine)
pip install -r backend/requirements-paddle.txt

# Formula recognition for scanned equations
pip install pix2tex
```

Force an engine with `PARSE_OCR_ENGINE=paddle|tesseract`.

---

## Testing

```bash
cd backend && python -m pytest -q
```

The suite (100+ tests) covers the API, pipeline, OCR, tables, charts, equations, search, summary, timing, platform detection, and the DOCX, PPTX and XLSX handlers. Test fixtures are generated programmatically by `tests/make_fixtures.py`.

---

## Project structure

```
Data-Parser/
├── backend/
│   ├── app/
│   │   ├── main.py               FastAPI app and router wiring
│   │   ├── api/                  upload, parse, jobs, search, export endpoints
│   │   ├── core/                 configuration, logging
│   │   ├── pipeline/             detector, router, assembler, confidence,
│   │   │                         provenance, failsafe, timing, orchestrator
│   │   ├── extractors/
│   │   │   ├── text/             digital text, reading order
│   │   │   ├── ocr/              OCR engine and image preprocessing
│   │   │   ├── tables/           detection and extraction
│   │   │   ├── charts/           detection and value extraction
│   │   │   ├── equations/        geometry-to-LaTeX
│   │   │   └── figures/          image and vector figure capture
│   │   ├── formats/              docx, pptx, xlsx, image handlers
│   │   ├── models/               Document, Block, Table, Search schemas
│   │   ├── output/               JSON and Markdown builders
│   │   ├── search/               search engine
│   │   ├── summary/              extractive summariser
│   │   └── utils/                bbox, file, platform helpers
│   ├── tests/
│   └── requirements*.txt
├── frontend/
│   └── src/                      components, pages, API service
├── data/
│   ├── uploads/                  incoming files
│   └── processed/<id>/           document.json, document.md, figures/
├── models/                       local model cache
└── .env.example
```

---

## Known limitations

- **Charts:** vector charts only; horizontal, stacked, log and dual-axis charts are preserved and flagged rather than extracted.
- **Equations:** LaTeX needs a text layer; scanned equations require `pix2tex`.
- **Scanned pages and images:** no deskew; Hindi and Tamil accuracy depends on installed language packs and scan quality.
- **DOCX / PPTX / XLSX:** no physical pagination; coordinates are never invented.

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| Scanned pages return an OCR error | Install Tesseract and confirm `tesseract --version` works, or set `TESSERACT_CMD`. |
| Hindi or Tamil text is missing or garbled | Run `tesseract --list-langs`; install `hin` or `tam` if absent. |
| `.env` values have no effect | The file is not auto-loaded. Export variables in the shell before starting the server. |
| Browser CORS errors | Set `CORS_ORIGINS` to the exact frontend origin. |
| Frontend cannot reach the API | Set `VITE_API_BASE` to your API URL including `/api`. |
| `413` on upload | Raise `PARSE_MAX_UPLOAD_MB`. |

---

## Repository

https://github.com/kshruthii/Data-Parser
