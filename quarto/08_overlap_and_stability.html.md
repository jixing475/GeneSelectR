---
title: "Overlap and Stability Analysis"
keep-md: true
execute:
  cache: true
  freeze: auto
  warning: false
  message: false
---





## Why Gene Set Overlap Matters

Different feature selection methods often produce **different gene lists**. This raises critical questions:

1. **Are the methods capturing the same biological signal?** (High overlap)
2. **Are they discovering complementary patterns?** (Low overlap, but both predictive)
3. **Is the signal robust or method-specific?** (Stability analysis)

GeneSelectR provides tools to quantify and visualize these relationships.

## Overlap Coefficients

### Jaccard Index

Measures set similarity:

$$
J(A, B) = \frac{|A \cap B|}{|A \cup B|}
$$

- Range: [0, 1]
- 0 = No overlap
- 1 = Perfect overlap

**Limitation**: Sensitive to set size. Two methods selecting 10 and 100 genes with 10 in common have J = 10/100 = 0.1.

### Overlap Coefficient (Szymkiewicz-Simpson)

$$
O(A, B) = \frac{|A \cap B|}{\min(|A|, |B|)}
$$

- Range: [0, 1]
- Normalizes by smaller set size
- Answers: "What fraction of the smaller set is contained in the larger?"

### Example Calculation




::: {.cell}

```{.r .cell-code}
set_A <- c("TP53", "EGFR", "BRCA1", "MYC")
set_B <- c("TP53", "EGFR", "KRAS")

# Jaccard
intersection <- length(intersect(set_A, set_B))
union <- length(union(set_A, set_B))
jaccard <- intersection / union
cat("Jaccard:", jaccard, "\n")
```

::: {.cell-output .cell-output-stdout}

```
Jaccard: 0.4 
```


:::

```{.r .cell-code}
# Overlap coefficient
min_size <- min(length(set_A), length(set_B))
overlap_coef <- intersection / min_size
cat("Overlap:", overlap_coef, "\n")
```

::: {.cell-output .cell-output-stdout}

```
Overlap: 0.6666667 
```


:::
:::




## Computing Overlap Matrices

### Using `calculate_overlap_coefficients()`




::: {.cell}

```{.r .cell-code}
suppressPackageStartupMessages({
  library(dplyr)
  library(ggplot2)
})

project_root <- "/Users/zero/Desktop/GeneSelectR"
annotated <- readRDS(file.path(project_root, "tests/testthat/fixtures/AnnotatedGeneLists.rds"))
```
:::




Extract gene symbols per method:




::: {.cell}

```{.r .cell-code}
gene_lists <- lapply(annotated@inbuilt, function(gl) gl@SYMBOL)
names(gene_lists)
```

::: {.cell-output .cell-output-stdout}

```
[1] "Lasso"        "Univariate"   "RandomForest" "boruta"       "background"  
[6] "DEG_rural"    "DEG_urban"   
```


:::
:::




Compute overlap matrix:




::: {.cell}

```{.r .cell-code}
library(GeneSelectR)

overlap_matrix <- calculate_overlap_coefficients(
  gene_lists,
  coefficient = "overlap"  # or "jaccard"
)

# Result: symmetric matrix of pairwise overlaps
overlap_matrix
```
:::




## Visualization

### Heatmap of Overlaps




::: {.cell}

```{.r .cell-code}
plot_overlap_heatmaps(
  overlap_matrix,
  title = "Gene Set Overlap (Inbuilt Importance)",
  color_palette = "Blues"
)
```
:::




Expected output: Heatmap where darker colors indicate higher overlap.

**Interpretation**:

- Diagonal = 1 (perfect self-overlap)
- Off-diagonal values reveal method concordance
- Clusters of high overlap suggest methods capturing similar signals

### UpSet Plot

For more than 3 sets, UpSet plots show intersection sizes:




::: {.cell}

```{.r .cell-code}
plot_upset(
  gene_lists,
  nsets = length(gene_lists),
  order.by = "freq"
)
```
:::




Expected output: Bar plot showing:

- Unique genes per method
- Pairwise intersections
- Higher-order intersections (e.g., genes in all 3 methods)

**Interpretation**:

- Large exclusive sets suggest method-specific discoveries
- Large intersections suggest consensus genes

## Stability Analysis

### Concept

**Stability** measures feature selection consistency across **perturbed datasets** (e.g., bootstrap samples).

Highly stable methods select similar features regardless of sampling variability; unstable methods are sensitive to data perturbations.

### Stability Metrics

1. **Jaccard stability**: Average Jaccard index across bootstrap pairs
2. **Pairwise overlap**: Overlap between selections on different samples

### Running Stability Analysis

GeneSelectR includes `inst/extras/` scripts for bootstrap-based stability (not executed in vignettes to save time):




::: {.cell}

```{.r .cell-code}
# Pseudocode (actual script at inst/extras/stability_analysis.R)
bootstrap_results <- lapply(1:100, function(i) {
  # Resample data with replacement
  sample_idx <- sample(1:nrow(X), replace = TRUE)
  X_boot <- X[sample_idx, ]
  y_boot <- y[sample_idx]
  
  # Run feature selection
  results <- perform_grid_search(X_boot, y_boot, ...)
  
  # Extract selected features
  top_features <- names(results@inbuilt_feature_importance$Lasso[1:50])
  return(top_features)
})

# Compute pairwise Jaccard
stability_scores <- combn(1:100, 2, function(idx) {
  length(intersect(bootstrap_results[[idx[1]]], bootstrap_results[[idx[2]]])) /
    length(union(bootstrap_results[[idx[1]]], bootstrap_results[[idx[2]]]))
})

mean(stability_scores)  # Overall stability
```
:::




### Interpreting Stability

- **High stability (>0.8)**: Robust gene signature, reliable across samples
- **Medium stability (0.5-0.8)**: Moderate robustness, validate with external data
- **Low stability (<0.5)**: Unstable selection, consider:
  - Increasing sample size
  - Reducing feature space (correlation filtering)
  - Using ensemble methods

## Design Rationale

### Why Multiple Overlap Metrics?

- **Jaccard**: Standard, symmetric, penalizes size differences
- **Overlap coefficient**: Favors smaller sets, useful when comparing methods with different selection cardinalities

Reporting both provides a fuller picture.

### Why Stability Matters

High predictive performance does **not** guarantee biological validity:

- **Overfitting**: Model memorizes noise, selecting spurious features
- **Data-specific patterns**: Features unique to the dataset, not generalizable

Stability testing via resampling identifies robust signals.

### Why Not Always Require High Overlap?

Low overlap between methods can indicate:

1. **Complementary signals**: Methods capture different aspects of biology (good!)
2. **Instability**: Both selections are noisy (bad)

Distinguish via external validation (e.g., literature support, pathway analysis).

## Example Workflow




::: {.cell}

```{.r .cell-code}
# 1. Run feature selection with multiple methods
results <- perform_grid_search(X, y, ...)

# 2. Annotate gene lists
annotated <- annotate_gene_lists(results, species = "Homo.sapiens")

# 3. Extract symbols per method
gene_lists_inbuilt <- lapply(annotated@inbuilt, function(gl) gl@SYMBOL)
gene_lists_perm <- lapply(annotated@permutation, function(gl) gl@SYMBOL)

# 4. Compute overlaps (inbuilt)
overlap_inbuilt <- calculate_overlap_coefficients(gene_lists_inbuilt, "jaccard")

# 5. Compute overlaps (permutation)
overlap_perm <- calculate_overlap_coefficients(gene_lists_perm, "jaccard")

# 6. Visualize
plot_overlap_heatmaps(overlap_inbuilt, title = "Inbuilt Importance Overlap")
plot_overlap_heatmaps(overlap_perm, title = "Permutation Importance Overlap")

# 7. UpSet plot for consensus
plot_upset(gene_lists_inbuilt, nsets = 4, order.by = "freq")

# 8. Identify consensus genes
consensus <- Reduce(intersect, gene_lists_inbuilt)
cat("Consensus genes:", length(consensus), "\n")
```
:::




## Case Study: Interpreting Low Overlap

Suppose LASSO and Random Forest share only 20% of genes:

**Possible explanations**:

1. **Linear vs non-linear signals**: LASSO captures marginal effects; RF captures interactions
2. **Regularization differences**: Different penalty strengths
3. **Instability**: Both methods overfitting

**Next steps**:

- Check stability via bootstrap
- Compare GO enrichment (do both sets enrich similar pathways?)
- Validate on external dataset

## Summary

GeneSelectR's overlap and stability tools:

1. **Quantify** agreement between feature selection methods
2. **Visualize** overlaps via heatmaps and UpSet plots
3. **Assess** robustness via bootstrap resampling (optional)

The next chapter explores functional interpretation via GO enrichment.

---

**Next**: [GO Enrichment Analysis](09_go_enrichment.qmd)
