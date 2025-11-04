---
title: "End-to-End Demo: Reproducible Workflow"
keep-md: true
execute:
  cache: true
  freeze: auto
  warning: false
  message: false
---





## Scenario

**Goal**: Identify gene signatures distinguishing two biological conditions using a subset of gene expression data.

**Data**: `UrbanRandomSubset` fixture containing:

- Expression matrix (samples × genes)
- Class labels (e.g., control vs. disease)

**Workflow**:

1. Environment setup
2. Data preprocessing
3. Feature selection with multiple methods
4. Performance evaluation
5. Feature importance extraction
6. Gene set overlap analysis
7. GO enrichment interpretation

This demo uses small fixture data to ensure fast rendering. For production analyses, scale up sample sizes and feature counts.

## Step 1: Environment Setup

### Load Required Packages




::: {.cell}

```{.r .cell-code}
suppressPackageStartupMessages({
  library(GeneSelectR)
  library(tidyverse)
  library(reticulate)
})

# Set seed for reproducibility
set.seed(42)
```
:::




### Configure Python Environment




::: {.cell}

```{.r .cell-code}
# Point to uv-managed venv (adjust path as needed)
project_root <- "/Users/zero/Desktop/GeneSelectR"
venv_path <- file.path(project_root, ".venv")

use_python(file.path(venv_path, "bin", "python"), required = TRUE)
py_config()
```
:::




**Note**: In practice, run this once per session. For rendering, assume environment is pre-configured.

### Import Python Modules




::: {.cell}

```{.r .cell-code}
import_python_packages()
modules <- define_sklearn_modules()
```
:::




## Step 2: Load and Inspect Data




::: {.cell}

```{.r .cell-code}
# Load fixture
project_root <- "/Users/zero/Desktop/GeneSelectR"
data_path <- file.path(project_root, "tests/testthat/fixtures/UrbanRandomSubset.rda")
load(data_path)

# Extract components (first column is treatment/group, rest are features)
y <- UrbanRandomSubset[, 1]
X <- as.matrix(UrbanRandomSubset[, -1])

# Convert character matrix to numeric
for(i in 1:ncol(X)) {
  X[, i] <- as.numeric(X[, i])
}

# Dimensions
cat("Samples:", nrow(X), "\n")
cat("Features:", ncol(X), "\n")
cat("Classes:", table(y), "\n")
```
:::




### Data Structure




::: {.cell}

```{.r .cell-code}
# First 5 samples, first 5 features
X[1:5, 1:5]

# Class distribution
barplot(table(y), col = c("skyblue", "salmon"), 
        main = "Sample Distribution", xlab = "Class", ylab = "Count")
```
:::




### Train-Test Split




::: {.cell}

```{.r .cell-code}
# 70-30 split
n <- nrow(X)
train_idx <- sample(1:n, size = 0.7 * n)

X_train <- X[train_idx, ]
y_train <- y[train_idx]
X_test <- X[-train_idx, ]
y_test <- y[-train_idx]

cat("Training samples:", nrow(X_train), "\n")
cat("Test samples:", nrow(X_test), "\n")
```
:::




## Step 3: Configure Feature Selection

### Set Parameters




::: {.cell}

```{.r .cell-code}
max_features <- 50  # Maximum genes to select

# Define preprocessing and feature selection methods
fs_setup <- set_default_fs_methods(modules, max_features, random_state = 42)

preprocessing_steps <- fs_setup$preprocessing_steps
fs_methods <- fs_setup$default_feature_selection_methods

# Define parameter grids
param_grids <- set_default_param_grids(max_features)
```
:::




### Create Pipelines




::: {.cell}

```{.r .cell-code}
# Classifier for evaluation
classifier <- modules$GradBoost(random_state = 42)

# Build pipelines
pipelines <- create_pipelines(
  preprocessing_steps = preprocessing_steps,
  fs_methods = fs_methods,
  classifier = classifier,
  modules = modules
)

cat("Pipelines created:", names(pipelines), "\n")
```
:::




## Step 4: Run Grid Search

**Note**: Set `eval=FALSE` for quick rendering; run interactively for real results.




::: {.cell}

```{.r .cell-code}
results <- perform_grid_search(
  pipelines = pipelines,
  param_grids = param_grids,
  X = X_train,
  y = y_train,
  cv = 3,  # 3-fold CV (faster for demo)
  scoring = "f1_weighted",
  random_state = 42,
  calculate_permutation_importance = TRUE,  # Include permutation importance
  n_jobs = 2  # Limit parallelism for demo
)

# Save results for reproducibility
saveRDS(results, "demo_results.rds")
```
:::




For this demo, use pre-computed results:




::: {.cell}

```{.r .cell-code}
results <- readRDS(file.path(project_root, "tests/testthat/fixtures/PipelineResults.rds"))
```
:::




## Step 5: Evaluate Performance

### Cross-Validation Scores




::: {.cell}

```{.r .cell-code}
cv_scores <- results@cv_mean_score %>%
  arrange(desc(mean_score))

print(cv_scores)
```
:::




**Best method**: (see results above) with highest mean score.

### Test Set Evaluation




::: {.cell}

```{.r .cell-code}
# Evaluate on held-out test set
test_metrics <- evaluate_test_metrics(results, X_test, y_test)

# Visualize
plot_metrics(results, metric_type = "test", color_palette = "Set1")
```
:::




## Step 6: Feature Importance

### Extract Top Features




::: {.cell}

```{.r .cell-code}
# Top 20 features (inbuilt importance)
best_method <- cv_scores$method[1]

if (!is.null(results@inbuilt_feature_importance[[best_method]])) {
  top_features <- results@inbuilt_feature_importance[[best_method]] %>%
    sort(decreasing = TRUE) %>%
    head(20)
  
  cat("Top 20 features for", best_method, ":\n")
  print(names(top_features))
} else {
  cat("Inbuilt importance not available\n")
}
```
:::




### Visualize Importance




::: {.cell}

```{.r .cell-code}
plot_feature_importance(
  results,
  method = best_method,
  importance_type = "inbuilt",
  top_n = 20,
  color = "steelblue"
)
```
:::




### Compare Inbuilt vs Permutation




::: {.cell}

```{.r .cell-code}
# If permutation importance was computed
if (length(results@permutation_importance) > 0) {
  p1 <- plot_feature_importance(results, best_method, "inbuilt", top_n = 15)
  p2 <- plot_feature_importance(results, best_method, "permutation", top_n = 15)
  
  library(patchwork)
  p1 + p2 + plot_layout(ncol = 2)
}
```
:::




## Step 7: Gene Set Overlap

### Extract Gene Lists




::: {.cell}

```{.r .cell-code}
# Annotate gene lists (requires org.db packages)
annotated <- annotate_gene_lists(
  results,
  species = "Homo.sapiens",
  keytype = "ENSEMBL",
  top_n = 50
)

# Gene lists per method (symbols)
gene_lists <- lapply(annotated@inbuilt, function(gl) gl@SYMBOL)
```
:::




For demo, use pre-annotated fixture:




::: {.cell}

```{.r .cell-code}
annotated <- readRDS(file.path(project_root, "tests/testthat/fixtures/AnnotatedGeneLists.rds"))
gene_lists <- lapply(annotated@inbuilt, function(gl) gl@SYMBOL)

# Number of genes per method
sapply(gene_lists, length)
```
:::




### Compute Overlap




::: {.cell}

```{.r .cell-code}
overlap_matrix <- calculate_overlap_coefficients(
  gene_lists,
  coefficient = "jaccard"
)

# Heatmap
plot_overlap_heatmaps(
  overlap_matrix,
  title = "Gene Set Overlap (Jaccard Index)",
  color_palette = "YlOrRd"
)
```
:::




### UpSet Plot




::: {.cell}

```{.r .cell-code}
plot_upset(
  gene_lists,
  nsets = length(gene_lists),
  order.by = "freq"
)
```
:::




**Interpretation**: Identify consensus genes selected by multiple methods.

## Step 8: GO Enrichment

### Define Background




::: {.cell}

```{.r .cell-code}
# Background = all features assayed
background <- colnames(X)
```
:::




For demo, use pre-defined background:




::: {.cell}

```{.r .cell-code}
background <- readRDS(file.path(project_root, "tests/testthat/fixtures/background.rds"))
```
:::




### Run Enrichment




::: {.cell}

```{.r .cell-code}
# Enrichment for best method
best_genes <- annotated@inbuilt[[best_method]]@ENTREZID

enrichment <- GO_enrichment_analysis(
  genes = best_genes,
  background = background,
  ont = "BP",  # Biological Process
  pAdjustMethod = "BH",
  qvalueCutoff = 0.05
)

# Top enriched terms
head(enrichment@result[, c("Description", "pvalue", "p.adjust", "Count")])
```
:::




### Visualize Enrichment




::: {.cell}

```{.r .cell-code}
# Dot plot
dotplot(enrichment, showCategory = 20, font.size = 10)

# Bar plot
barplot(enrichment, showCategory = 15)
```
:::




### Simplify Redundant Terms




::: {.cell}

```{.r .cell-code}
simplified <- run_simplify_enrichment(
  enrichment,
  measure = "Wang",
  threshold = 0.7
)

# Representative terms
head(simplified$Description)
```
:::




## Step 9: Reporting and Export

### Summary Statistics




::: {.cell}

```{.r .cell-code}
cat("=== Analysis Summary ===\n")
cat("Dataset:", "UrbanRandomSubset\n")
cat("Training samples:", nrow(X_train), "\n")
cat("Test samples:", nrow(X_test), "\n")
cat("Features:", ncol(X), "\n")
cat("Best method:", "(see results above)", "\n")
cat("Best CV score:", "(see results above)", "\n")
cat("Selected features:", length(top_features), "\n")
```
:::




### Export Results




::: {.cell}

```{.r .cell-code}
# Save results
saveRDS(results, "pipeline_results.rds")
saveRDS(annotated, "annotated_gene_lists.rds")

# Export gene lists
for (method in names(gene_lists)) {
  write.csv(
    data.frame(Gene = gene_lists[[method]]),
    file = paste0(method, "_genes.csv"),
    row.names = FALSE
  )
}

# Export enrichment
write.csv(enrichment@result, "go_enrichment.csv", row.names = FALSE)
```
:::




## Design Decisions Recap

### Why Multiple Methods?

Different algorithms capture different patterns:

- **LASSO**: Linear, sparse
- **Random Forest**: Non-linear, interactions
- **Univariate**: Fast, marginal effects
- **Boruta**: Comprehensive, all-relevant

Consensus across methods increases confidence.

### Why Cross-Validation?

- Estimates generalization performance
- Reduces overfitting risk
- Provides uncertainty quantification (std)

### Why GO Enrichment?

- Translates gene lists into biological insights
- Validates findings against established knowledge
- Guides follow-up experiments

## Next Steps (Beyond Demo)

1. **Increase sample size**: More robust signatures
2. **External validation**: Test on independent cohorts
3. **Stability analysis**: Bootstrap resampling
4. **Pathway analysis**: KEGG, Reactome
5. **Network analysis**: Protein-protein interactions
6. **Clinical validation**: Experimental confirmation

## Summary

This end-to-end workflow demonstrates:

1. **Environment setup** with uv and reticulate
2. **Pipeline creation** with multiple feature selection methods
3. **Hyperparameter optimization** via grid search
4. **Performance evaluation** on CV and test sets
5. **Feature importance** extraction and visualization
6. **Gene set overlap** analysis
7. **GO enrichment** for biological interpretation

All steps are reproducible via Quarto caching and fixed random seeds.

---

**Next**: [Synthesis: Design and Extensions](11_synthesis.qmd)
