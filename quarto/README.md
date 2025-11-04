# GeneSelectR Quarto Website

This directory contains a narrative, literate programming-style documentation website for GeneSelectR.

## Overview

The website provides a chapterized tour of the package design, implementation, and usage:

1. **01_overview.qmd** - High-level architecture and entry points
2. **02_setup_uv_reticulate.qmd** - Python environment setup with uv
3. **03_core_classes.qmd** - S4 class definitions and semantics
4. **04_python_integration.qmd** - R-Python bridge via reticulate
5. **05_feature_selection_and_search.qmd** - Pipeline creation and grid search
6. **06_metrics_and_visualization.qmd** - Performance metrics and plots
7. **07_feature_importance.qmd** - Inbuilt and permutation importance
8. **08_overlap_and_stability.qmd** - Gene set comparison and stability
9. **09_go_enrichment.qmd** - Functional enrichment analysis
10. **10_end_to_end_demo.qmd** - Complete reproducible workflow
11. **11_synthesis.qmd** - Design reflections and extensions

## Prerequisites

### 1. Install Quarto

```bash
# macOS
brew install quarto

# Or download from https://quarto.org/docs/get-started/
```

### 2. Set Up Python Environment (Optional for viewing)

For executing code chunks (not required for basic rendering):

```bash
# From repository root
cd /Users/zero/Desktop/GeneSelectR

# Create virtual environment with uv
uv venv .venv
uv pip install -r inst/python/requirements.txt
```

### 3. Install R Dependencies

```r
# In R
install.packages(c("reticulate", "tidyverse", "ggplot2"))

# Bioconductor packages
if (!requireNamespace("BiocManager", quietly = TRUE))
    install.packages("BiocManager")
BiocManager::install(c("clusterProfiler", "org.Hs.eg.db"))
```

## Rendering the Website

### Full Site Render

From the repository root:

```bash
quarto render quarto/
```

Or from within the `quarto/` directory:

```bash
cd quarto
quarto render
```

Output will be in `quarto/_site/`.

### Preview During Development

Live preview with auto-reload:

```bash
quarto preview quarto/
```

Opens browser at `http://localhost:4200`.

### Render Single Chapter

```bash
quarto render quarto/01_overview.qmd
```

## Key Features

### Caching

All documents use `cache: true` and `freeze: auto` to preserve computation results:

- First render executes all code chunks
- Subsequent renders reuse cached results (unless code changes)
- Prevents expensive recomputation during documentation updates

### Keep Markdown

Every `.qmd` file includes `keep-md: true`, preserving intermediate Markdown files alongside HTML output.

### Fixtures

Code examples use small test fixtures from `tests/testthat/fixtures/` to ensure fast rendering:

- `UrbanRandomSubset.rda` - Sample expression data
- `PipelineResults.rds` - Pre-computed pipeline results
- `AnnotatedGeneLists.rds` - Annotated gene lists
- `EnrichGO.rds` - GO enrichment results
- `background.rds` - Background gene set

Heavy computations are set to `eval=FALSE` with instructions for enabling.

## Project Guarantees

✅ **No source modifications**: This website does not alter any R package source files (`R/`, `inst/python/`, `NAMESPACE`, etc.)

✅ **Independent environment**: Python dependencies managed via uv, bound via reticulate per-session

✅ **Reproducible**: Fixed random seeds, locked dependencies, and Quarto caching ensure consistent results

## Customization

### Changing Theme

Edit `_quarto.yml`:

```yaml
format:
  html:
    theme: darkly  # Try: cosmo, flatly, journal, lumen, etc.
```

### Adding New Chapters

1. Create `XX_new_chapter.qmd` with required YAML front matter:

```yaml
---
title: "Your Title"
keep-md: true
execute:
  cache: true
  freeze: auto
  warning: false
  message: false
---
```

2. Add to `_quarto.yml` navbar:

```yaml
website:
  navbar:
    left:
      - text: New Chapter
        href: XX_new_chapter.qmd
```

### Citations

Add entries to `references.bib` and cite with `[@citation_key]`.

## Troubleshooting

### "Quarto command not found"

Install Quarto from https://quarto.org/docs/get-started/

### "Python module not found"

Ensure the uv virtual environment is activated or configure reticulate:

```r
library(reticulate)
use_python("/path/to/.venv/bin/python", required = TRUE)
```

### Slow rendering

Use `quarto render --cache-refresh` to force cache rebuild if stale results are suspected.

### Code chunks not executing

Check that:
1. R packages are installed
2. Python environment is configured (if needed)
3. Fixtures exist in `tests/testthat/fixtures/`

## Contributing

To contribute:

1. Edit `.qmd` files (not generated HTML)
2. Test with `quarto preview`
3. Verify `keep-md: true` is present in YAML
4. Ensure code chunks use fixtures for speed
5. Submit pull request with rendered site

## License

Same as GeneSelectR package (see repository root LICENSE file).
