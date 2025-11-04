---
title: "Feature Importance Estimation"
keep-md: true
execute:
  cache: true
  freeze: auto
  warning: false
  message: false
---





## Two Flavors of Importance

GeneSelectR extracts feature importance via two approaches:

1. **Inbuilt (model-based)**: Importance scores from the ML algorithm itself
   - Fast (no extra computation)
   - Method-specific (e.g., Gini importance for Random Forest)
   - May be biased by feature correlations

2. **Permutation-based**: Importance via feature shuffling
   - Slower (requires retraining)
   - Model-agnostic (works for any algorithm)
   - More robust to correlated features

Both are stored in the `PipelineResults` object after grid search.

## Inbuilt Feature Importance

### Extraction from Models

After training, scikit-learn models expose importance via:

- **Tree-based** (Random Forest, Gradient Boosting): `feature_importances_` attribute
- **Linear models** (LASSO): Absolute coefficient values
- **Univariate**: Statistical test scores
- **Boruta**: Binary support mask

GeneSelectR normalizes these to a common scale (sum to 1).

### Accessing Inbuilt Importances




::: {.cell}

```{.r .cell-code}
suppressPackageStartupMessages({
  library(dplyr)
  library(ggplot2)
})

project_root <- "/Users/zero/Desktop/GeneSelectR"
results <- readRDS(file.path(project_root, "tests/testthat/fixtures/PipelineResults.rds"))
```
:::




Raw importances (named numeric vectors):




::: {.cell}

```{.r .cell-code}
# Example: Lasso importances
lasso_imp <- results@inbuilt_feature_importance$Lasso

# Check if it's a vector or data.frame
if (is.data.frame(lasso_imp) || is.list(lasso_imp)) {
  cat("Lasso importance structure:\n")
  str(lasso_imp, max.level = 1)
} else {
  # It's a vector, can sort directly
  head(sort(lasso_imp, decreasing = TRUE), 10)
}
```

::: {.cell-output .cell-output-stdout}

```
Lasso importance structure:
tibble [33 × 8] (S3: tbl_df/tbl/data.frame)
```


:::
:::




Higher values indicate greater importance for classification.

### Aggregation Across CV Folds

`get_feature_importances()` averages importances across CV splits:




::: {.cell}

```{.r .cell-code}
top_features <- get_feature_importances(
  results,
  method = "Lasso",
  importance_type = "inbuilt",
  top_n = 50  # Return top 50 features
)

head(top_features)
```
:::




Output is a data.frame with columns `feature`, `importance`, `rank`.

## Permutation Feature Importance

### Concept

Permutation importance measures the **drop in model performance** when a feature is randomly shuffled:

1. Compute baseline score on validation set
2. Permute feature *i* (break feature-target relationship)
3. Re-compute score
4. Importance = baseline - permuted score

Features causing large drops are important; features causing no change are irrelevant.

### Advantages Over Inbuilt

- **Model-agnostic**: Works for any classifier
- **Correlation-aware**: Doesn't inflate correlated features
- **Interpretable**: Directly tied to predictive performance

### Computational Cost

Permutation importance requires:

- One prediction per feature per permutation (default: 10 permutations)
- For 1000 features: 10,000 predictions

Still faster than retraining, but slower than inbuilt.

### Accessing Permutation Importances




::: {.cell}

```{.r .cell-code}
# Check if permutation importances were computed
if (length(results@permutation_importance) > 0) {
  lasso_perm <- results@permutation_importance$Lasso
  head(lasso_perm)
} else {
  cat("Permutation importance not computed (set calculate_permutation_importance = TRUE)\n")
}
```

::: {.cell-output .cell-output-stdout}

```
# A tibble: 6 × 8
  feature          mean_importance     std rank_Lasso_split_1 rank_Lasso_split_2
  <chr>                      <dbl>   <dbl>              <int>              <int>
1 ENSG00000071539         0.000656 1.47e-3                  1               1121
2 ENSG00000134690         0.00197  4.40e-3               6480                  3
3 ENSG00000171848         0.00164  3.67e-3              12042                  5
4 ENSG00000175305         0.000328 7.33e-4              12611                  7
5 ENSG00000186193…        0.000984 2.20e-3              14309                  6
6 ENSG00000287914…        0.00230  1.27e-2              25858                  1
# ℹ 3 more variables: rank_Lasso_split_3 <int>, rank_Lasso_split_4 <int>,
#   rank_Lasso_split_5 <int>
```


:::
:::




Columns:

- `feature`: Feature name
- `importance_mean`: Mean importance across permutations
- `importance_std`: Standard deviation across permutations

## Visualization

### Bar Plots




::: {.cell}

```{.r .cell-code}
library(GeneSelectR)

plot_feature_importance(
  results,
  method = "Lasso",
  importance_type = "inbuilt",
  top_n = 20,
  color = "steelblue"
)
```
:::




Expected output: Horizontal bar plot of top 20 features.

### Comparing Inbuilt vs Permutation




::: {.cell}

```{.r .cell-code}
# Inbuilt
p1 <- plot_feature_importance(results, "Lasso", "inbuilt", top_n = 15)

# Permutation
p2 <- plot_feature_importance(results, "Lasso", "permutation", top_n = 15)

# Side-by-side
library(patchwork)
p1 + p2 + plot_layout(ncol = 2)
```
:::




**Observation**: Inbuilt and permutation rankings often differ. Permutation may downrank correlated features.

### Heatmap Across Methods




::: {.cell}

```{.r .cell-code}
# Extract top 20 features per method
importance_matrix <- sapply(names(results@inbuilt_feature_importance), function(method) {
  imp <- results@inbuilt_feature_importance[[method]]
  top_features <- names(sort(imp, decreasing = TRUE)[1:20])
  imp[top_features]
})

# Heatmap
pheatmap::pheatmap(
  importance_matrix,
  cluster_rows = TRUE,
  cluster_cols = FALSE,
  main = "Feature Importance Across Methods"
)
```
:::




Reveals method-specific preferences (e.g., LASSO favors different genes than Random Forest).

## Feature Selection Strategies

### Top-N Selection

Select the *N* most important features:




::: {.cell}

```{.r .cell-code}
top_genes <- names(sort(lasso_imp, decreasing = TRUE)[1:50])
```
:::




**Pro**: Simple, interpretable  
**Con**: Arbitrary threshold; may miss weakly important features

### Threshold-Based

Select features above an importance threshold:




::: {.cell}

```{.r .cell-code}
# Keep features with importance > 1% of max
threshold <- 0.01 * max(lasso_imp)
selected <- names(lasso_imp[lasso_imp > threshold])
```
:::




**Pro**: Adaptive to importance distribution  
**Con**: Threshold choice is subjective

### Cumulative Importance

Select features capturing *X%* of total importance:




::: {.cell}

```{.r .cell-code}
sorted_imp <- sort(lasso_imp, decreasing = TRUE)
cumsum_imp <- cumsum(sorted_imp) / sum(sorted_imp)
selected <- names(sorted_imp[cumsum_imp <= 0.90])  # Top 90%
```
:::




**Pro**: Ensures coverage of importance signal  
**Con**: May select many features in flat distributions

## Design Decisions

### Why Both Inbuilt and Permutation?

- **Inbuilt**: Fast screening for initial exploration
- **Permutation**: Validation of inbuilt rankings; less biased

Discrepancies between the two warrant investigation (e.g., multicollinearity).

### Why Normalize Importances?

Raw importances have method-specific scales:

- Random Forest: Sum of Gini decreases
- LASSO: Absolute coefficient values

Normalization to [0, 1] allows cross-method comparison.

### Why Average Across CV Folds?

Feature importance can vary across folds due to:

- Different training samples
- Different hyperparameters (in nested CV)

Averaging stabilizes rankings.

## Troubleshooting

### "All importances are zero"

For LASSO, this happens when regularization is too strong:




::: {.cell}

```{.r .cell-code}
# Check C parameter (lower = more regularization)
results@best_pipeline$params$feature_selector__estimator__C

# Solution: Increase C in param_grids
param_grids$Lasso$feature_selector__estimator__C <- c(1, 10, 100)
```
:::




### "Importances don't match selected features"

Feature selection happens during grid search; importances are extracted post-hoc. They may not align if:

- Threshold applied during selection
- Different hyperparameters between selection and importance extraction

Always cross-check with the actual selected features.

## Example Workflow




::: {.cell}

```{.r .cell-code}
# 1. Run grid search with permutation importance
results <- perform_grid_search(
  X, y,
  calculate_permutation_importance = TRUE,
  ...
)

# 2. Extract top features (inbuilt)
top_inbuilt <- get_feature_importances(results, "Lasso", "inbuilt", top_n = 50)

# 3. Extract top features (permutation)
top_perm <- get_feature_importances(results, "Lasso", "permutation", top_n = 50)

# 4. Compare overlaps
length(intersect(top_inbuilt$feature, top_perm$feature))  # How many in common?

# 5. Visualize
plot_feature_importance(results, "Lasso", "permutation", top_n = 20)

# 6. Annotate for GO analysis
annotated <- annotate_gene_lists(
  top_perm$feature,
  species = "Homo.sapiens",
  keytype = "ENSEMBL"
)
```
:::




## Summary

GeneSelectR's importance framework:

1. **Extracts** inbuilt importances from ML models (fast)
2. **Computes** permutation importances for robustness (slower)
3. **Aggregates** across CV folds for stability
4. **Visualizes** via bar plots and heatmaps

The next chapter examines gene set overlap and stability.

---

**Next**: [Overlap and Stability Analysis](08_overlap_and_stability.qmd)
