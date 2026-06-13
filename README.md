# V² — Variant Viewer

A lightweight, zero-dependency variant viewer that runs entirely in the browser. Load a VCF file and explore variants through a table and an interactive dashboard of charts.

---

## Features

- **VCF file parsing** — drag-and-drop or upload any standard `.vcf` / `.txt` VCF file
- **Virtual-scroll table** — handles millions of variants
- **Dashboard view** — donut chart (variant types) and bar charts (clinical significance, allele frequency buckets, chromosomes)
- **Filters** — chromosome, classification, and variant type dropdowns; live gene/ID search
- **Sort** — click any column header to sort ascending/descending
- **Detail panel** — click any row to open a slide-in panel with full variant info, raw INFO field, and per-sample genotype table (GT, DS)

---

## Usage

Open `variant-viewer.html` in any modern browser. No build step required.

```
# Option 1: just open the file
open variant-viewer.html

# Option 2: serve locally (avoids any browser file:// restrictions)
python3 -m http.server 8080
# then visit http://localhost:8080/variant-viewer.html
```

### Loading data

1. Click **Upload VCF** or **Load Demo** from the empty state screen
2. Switch between **Table View** and **Dashboard** using the sidebar tabs
3. Use the search box and dropdowns to filter; click any row to inspect the full variant

---

## VCF Format Support

The parser handles standard 8-column VCF and multi-sample VCF (columns 9+).

| Field | How it's used |
|---|---|
| `CHROM` | Chromosome (with or without `chr` prefix) |
| `POS` | Genomic position |
| `ID` | Used as gene name fallback if `GENEINFO`/`GENE` not in INFO |
| `REF` / `ALT` | Determines variant type: single-base same-length = SNV, otherwise INDEL |
| `QUAL` / `FILTER` | Displayed in detail panel |
| `INFO` | Parsed for `AF=`, `GENE=`, `GENEINFO=`, and classification keywords |
| `FORMAT` + samples | GT and DS extracted per sample; shown in detail panel table |

**Classification inference** — the tool reads free-text keywords from the INFO field (`pathogenic`, `likely_pathogenic`, `benign`, `likely_benign`). Anything unmatched defaults to `VUS`. For annotated VCFs (e.g. ClinVar exports), this hopefully works out of the box.

---

## Project Structure

```
variant-viewer.html    # entire application — HTML, CSS, JS in one file
README.md
DEV_NOTES.md
```

Everything lives in a single HTML file intentionally. This makes it trivially shareable and deployable.

---

## Browser Support

Any modern browser with Canvas 2D and ES6 support. Tested in Chrome, Firefox, and Safari.

---

## Limitations

- Classification inference is keyword-based; it is **not** a substitute for validated clinical annotation pipelines
- Very large VCFs (>20M variants) will parse slowly on the main thread — a Web Worker refactor is in the works
- No offline annotation: AF, gene, and clinical data must already be in the VCF INFO field