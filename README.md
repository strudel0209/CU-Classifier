# Engineering diagram classifier — Azure Content Understanding

A Jupyter notebook ([diagram_classifier.ipynb](./diagram_classifier.ipynb)) that classifies an input image or document
page into one label, without returning the full content extraction:

| Label | Meaning |
|---|---|
| `SingleLineDiagram` | Electrical power-system drawing (power distribution topology and equipment) |
| `ControllerLogic` | Control / automation drawing (ladder logic, function blocks, interlocks, signal logic) |
| `Other` | Anything else (photos, P&IDs, plans, datasheets, tables, text, ...) |

It uses the [`azure-ai-contentunderstanding`](https://pypi.org/project/azure-ai-contentunderstanding/) Python SDK and
implements two classification approaches so they can be compared on the same inputs.

## Approaches

Both approaches use a custom analyzer built on `prebuilt-document` and the same category definitions (`CATEGORIES` in the notebook).

| | Option A — `classify` field | Option B — `contentCategories` classifier |
|---|---|---|
| Configuration | One enum field (`diagramType`) with `method: classify` | `contentCategories` with `enableSegment: false` |
| Output | `fields.diagramType` (label + confidence) | `segments[].category` (label only) |
| Figure description | Enabled (`enableFigureDescription`) | Not enabled |
| Typical use | Return a label only | Route each category to its own extraction analyzer by adding `analyzerId` to a category |

**Why a document analyzer and not `prebuilt-image`?** The image base analyzer supports only `returnDetails`;
`contentCategories`, `enableSegment` and `omitContent` are ignored there. Document analyzers accept image files, so images are sent to one.

**Note:** `omitContent` must be `false` when a field schema is defined (the service rejects `true`), and for Option B
the `segments` array lives on the content object.

## Prerequisites

- Python 3.10+
- An Azure AI Foundry resource with Content Understanding enabled
- A supported chat-completion model deployed and mapped as a Content Understanding default
  (see [supported models](https://learn.microsoft.com/azure/ai-services/content-understanding/service-limits))
- Authentication: either an API key, or Entra ID (`az login` / managed identity) through `DefaultAzureCredential`

## Setup

```bash
cp .env.sample .env      # then fill in the values
pip install -r requirements.txt
```

Open the notebook and run the cells in order.

### Configuration (`.env`)

| Variable | Purpose | Default |
|---|---|---|
| `CONTENTUNDERSTANDING_ENDPOINT` | Foundry resource endpoint | required |
| `CONTENTUNDERSTANDING_KEY` | API key. If empty, `DefaultAzureCredential` is used | empty |
| `CU_API_VERSION` | `2025-11-01` (GA) or `2026-06-01-preview` | `2025-11-01` |
| `CU_COMPLETION_MODEL` | Model **name** used by the analyzers | `gpt-5.2` |
| `CU_COMPLETION_DEPLOYMENT` | Deployment name; used only when `SET_DEFAULTS = True` | — |
| `CU_EMBEDDING_MODEL` / `CU_EMBEDDING_DEPLOYMENT` | Embedding model and deployment; used only when `SET_DEFAULTS = True` | — |
| `CU_ANALYZER_A` / `CU_ANALYZER_B` | Analyzer IDs (letters, digits, `.`, `_`) | `diagramTypeFieldClassifier` / `diagramTypeCategoryClassifier` |
| `SAMPLES_DIR` | Folder with test images | `./samples` |

The preview API version has no SLA and is not recommended for production workloads.

## Notebook walkthrough

1. **Configuration and client** — loads `.env` and creates a `ContentUnderstandingClient`. The SDK handles authentication,
   api-version and long-running-operation polling.
2. **Default model deployments (optional, one-off)** — set `SET_DEFAULTS = True` once to map model names to your deployment names,
   if not already done in Content Understanding Studio.
3. **Category definitions** — `CATEGORIES` maps each label to a description. The descriptions are the main lever on classification quality.
   Describe what each drawing type *is* and communicates, not traits of specific images, and keep an explicit `Other` class.
4. **Option A analyzer** — creates the `classify`-field analyzer (`create_analyzer` replaces an existing analyzer with the same ID).
5. **Option B analyzer** — creates the `contentCategories` analyzer.
6. **Analyze helpers** — `analyze_file` sends a local file as binary; `analyze_url` analyzes a file by URL (for example in Blob storage).
7. **Result parsers** — `label_from_field` (Option A) and `label_from_categories` (Option B) return `(label, confidence)`.
   Option B returns no confidence.
8. **Try multiple images** — runs both options on the files in `SAMPLES_DIR` (recursive) in parallel and shows one results table.
   `SAMPLE_LIMIT` limits the number of files (`None` = all) and `MAX_WORKERS` sets parallelism.
   Raw responses are cached in `raw` for the session, so already analyzed files are not re-analyzed. A failure on one file is recorded in an `error` column and does not stop the run.
9. **Inspect a raw response** — prints the raw Option A response of one file, useful when a label comes back empty.
10. **Batch run + accuracy** — reuses the cached results, adds accuracy and a confusion matrix per option when expected labels are available.
11. **Save results** — writes `classification_results.csv`.
12. **Clean up** — set `CLEANUP = True` to delete both analyzers from the resource.

## Test images and expected labels

Place images in sub-folders named after the expected label. The folder name is used as the ground truth for scoring;
the analyzers never see it.

```
samples/
  SingleLineDiagram/*.png
  ControllerLogic/*.png
  Other/*.jpg
```

- Folder names must match the labels exactly. Files in any other folder (or directly in `samples/`) are still classified but are not scored.
- Accuracy is computed per option as the share of labelled files where the predicted label equals the folder name.
- Small test sets give noisy numbers: with N images, each image moves accuracy by 100/N points. Evaluate category-definition changes
  on a representative labelled set rather than on individual images.

## Supported inputs

The notebook picks up `.png`, `.jpg`, `.jpeg`, `.tif`, `.tiff`, `.bmp`, `.pdf` and `.heif` files.
Content Understanding itself also accepts other document types; see
[service quotas and limits](https://learn.microsoft.com/azure/ai-services/content-understanding/service-limits).

- **CAD formats (`.dwg`, `.dxf`) are not supported.** Export or plot drawings to PDF or an image format first.
- Images: up to 200 MB, 50×50 to 10,000×10,000 pixels.
- Documents: the synchronous path processes up to 5 pages; use `content_range` (for example `"1-3"`) to choose pages.

## Output

For each analyzed file the results table contains:

| Column | Description |
|---|---|
| `file` | Path relative to `SAMPLES_DIR` |
| `expected` | Folder-derived label, or empty |
| `A_label`, `A_conf`, `A_sec` | Option A label, confidence and latency in seconds |
| `B_label`, `B_conf`, `B_sec` | Option B label, confidence (always empty) and latency in seconds |
| `error` | Error message, if the file failed |

Cached files keep the latency from the call that analyzed them.

## Extending

- **Route to extraction (Option B):** add `"analyzerId": "<extractor>"` to a category in `contentCategories`.
- **New label:** add an entry to `CATEGORIES`; both analyzers are rebuilt from it when their cells are re-run.
- **Preview API:** set `CU_API_VERSION=2026-06-01-preview` in `.env`, restart the kernel and re-run.

## Costs and cleanup

Each analyzed image makes one analyzer call per option (two with both options). Analyzers persist on the resource until deleted
(set `CLEANUP = True` in the last cell).
