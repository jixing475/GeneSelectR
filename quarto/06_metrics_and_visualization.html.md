---
title: "Metrics and Visualization"
keep-md: true
execute:
  cache: true
  freeze: auto
  warning: false
  message: false
---





## Performance Metrics Framework

GeneSelectR evaluates feature selection pipelines using standard classification metrics:

- **F1 Score**: Harmonic mean of precision and recall (balances false positives/negatives)
- **Accuracy**: Proportion of correct predictions
- **Precision**: Proportion of positive predictions that are correct
- **Recall**: Proportion of actual positives correctly identified
- **AUC-ROC**: Area under the receiver operating characteristic curve

These metrics are computed at two stages:

1. **Cross-validation**: On training folds (for hyperparameter selection)
2. **Test set**: On held-out data (for unbiased performance estimation)

## Cross-Validation Metrics

### Extracting CV Scores

After `perform_grid_search()`, results contain CV scores for all hyperparameter combinations:




::: {.cell}

```{.r .cell-code}
suppressPackageStartupMessages({
  library(dplyr)
  library(ggplot2)
})

project_root <- "/Users/zero/Desktop/GeneSelectR"
fixture_path <- file.path(project_root, "tests/testthat/fixtures/PipelineResults.rds")
results <- readRDS(fixture_path)
```
:::




Raw CV results (truncated for brevity):




::: {.cell}

```{.r .cell-code}
# Example: Lasso CV results
lasso_cv <- results@cv_results$Lasso
names(lasso_cv)  # Keys from sklearn's cv_results_ dict
```

::: {.cell-output .cell-output-stdout}

```
NULL
```


:::
:::




### Aggregated CV Scores

`calculate_mean_cv_scores()` computes mean and std across folds:




::: {.cell}

```{.r .cell-code}
cv_scores <- results@cv_mean_score

# Rank methods by mean score
cv_scores %>%
  arrange(desc(mean_score)) %>%
  select(method, mean_score, sd_score)
```

::: {.cell-output .cell-output-stdout}

```
        method mean_score    sd_score
1       boruta  0.9210374 0.012047342
2        Lasso  0.9153061 0.007984372
3 RandomForest  0.9127721 0.007926976
4   Univariate  0.9045578 0.011999833
```


:::
:::




**Interpretation**:

- **mean_score**: Average performance across CV folds (higher = better)
- **sd_score**: Variability across folds (lower = more stable)
- **rank**: Ranking among methods (1 = best)

## Test Set Metrics

### Evaluation on Held-Out Data

To avoid overestimating performance (CV leakage), evaluate on a separate test set:




::: {.cell}

```{.r .cell-code}
# Split data
train_idx <- sample(1:nrow(X), size = 0.7 * nrow(X))
X_train <- X[train_idx, ]
y_train <- y[train_idx]
X_test <- X[-train_idx, ]
y_test <- y[-train_idx]

# Fit on training, evaluate on test
results <- perform_grid_search(X_train, y_train, ...)
test_metrics <- evaluate_test_metrics(results, X_test, y_test)
```
:::




### Test Metrics Structure




::: {.cell}

```{.r .cell-code}
# From fixture (simulated test set)
test_metrics <- results@test_metrics

# Structure depends on evaluation mode
if (is.data.frame(test_metrics)) {
  head(test_metrics)
} else {
  cat("Test metrics is a list with", length(test_metrics), "elements\n")
}
```

::: {.cell-output .cell-output-stdout}

```
# A tibble: 4 × 9
  method       f1_mean  f1_sd recall_mean recall_sd precision_mean precision_sd
  <chr>          <dbl>  <dbl>       <dbl>     <dbl>          <dbl>        <dbl>
1 Lasso          0.854 0.0351       0.858    0.0388          0.870       0.0262
2 RandomForest   0.873 0.0346       0.876    0.0356          0.883       0.0286
3 Univariate     0.859 0.0127       0.863    0.0150          0.866       0.0136
4 boruta         0.876 0.0251       0.876    0.0273          0.889       0.0234
# ℹ 2 more variables: accuracy_mean <dbl>, accuracy_sd <dbl>
```


:::
:::




**Columns** (if data.frame):

- `method`: Feature selection method
- `f1_mean`, `f1_std`: F1 score (mean ± std across splits)
- `accuracy_mean`, `accuracy_std`: Accuracy
- `precision_mean`, `precision_std`: Precision
- `recall_mean`, `recall_std`: Recall

## Visualization with `plot_metrics()`

### CV Scores Plot




::: {.cell}

```{.r .cell-code}
# library(GeneSelectR)

p <- plot_metrics(
  results
)
print(p)
```
:::




Expected output: Bar plot of mean CV scores per method with error bars (std).

### Test Scores Plot




::: {.cell}

```{.r .cell-code}
p <- plot_metrics(
  results
)
print(p)
```
:::




Expected output: Grouped bar plot of F1, accuracy, precision, recall per method.

### Customization

`plot_metrics()` returns a ggplot object, allowing further customization:




::: {.cell}

```{.r .cell-code}
p <- plot_metrics(results)

p + 
  labs(title = "Cross-Validation Performance Comparison") +
  theme_minimal() +
  theme(axis.text.x = element_text(angle = 45, hjust = 1))
```
:::




## Interpreting Metrics

### F1 vs Accuracy

**When to prioritize F1**:

- Imbalanced classes (e.g., 90% control, 10% disease)
- False negatives/positives have unequal costs

**When to prioritize accuracy**:

- Balanced classes
- Equal cost for all error types

Example: Cancer diagnosis (imbalanced, high cost of false negatives) → Use F1.

### Precision vs Recall Trade-off

- **High precision**: Few false positives (conservative gene selection)
- **High recall**: Few false negatives (liberal gene selection)

GeneSelectR uses F1 (balanced) by default, but users can optimize for specific metrics.

### Variance Matters




::: {.cell}

```{.r .cell-code}
# Method A: mean=0.85, std=0.10 (unstable)
# Method B: mean=0.83, std=0.02 (stable)

# Prefer Method B if consistency is critical (e.g., clinical deployment)
```
:::




Low variance indicates robust feature selection across different data splits.

## Comparing Methods

### Statistical Significance

For rigorous comparison, use paired tests on CV folds:




::: {.cell}

```{.r .cell-code}
# Extract fold-level scores for two methods
lasso_scores <- results@cv_results$Lasso$split_test_score
rf_scores <- results@cv_results$RandomForest$split_test_score

# Paired t-test
t.test(lasso_scores, rf_scores, paired = TRUE)
```
:::




**Caution**: Multiple comparisons require correction (e.g., Bonferroni).

### Effect Size




::: {.cell}

```{.r .cell-code}
# Cohen's d
mean_diff <- mean(lasso_scores - rf_scores)
pooled_sd <- sqrt((var(lasso_scores) + var(rf_scores)) / 2)
cohens_d <- mean_diff / pooled_sd

# Interpretation: |d| > 0.8 = large effect
```
:::




## Design Considerations

### Why Multiple Metrics?

Single metrics can be misleading:

- **Accuracy paradox**: 95% accuracy on 95:5 class split achieved by always predicting majority class
- **F1 alone**: Ignores true negatives (less critical for gene selection but relevant for balanced reporting)

GeneSelectR reports all standard metrics for transparency.

### Why Report Standard Deviation?

Machine learning performance varies across:

- Random initialization
- Data splits
- Hyperparameter configurations

Reporting std quantifies this uncertainty, guiding method selection.

### Why Separate CV and Test Metrics?

- **CV metrics**: For hyperparameter tuning (biased estimate)
- **Test metrics**: For unbiased performance evaluation

Conflating the two leads to overoptimistic estimates.

## Example Workflow




::: {.cell}

```{.r .cell-code}
# 1. Run grid search with CV
results <- perform_grid_search(X_train, y_train, ...)

# 2. Compare methods via CV scores
cv_comparison <- results@cv_mean_score %>%
  arrange(desc(mean_score))

# 3. Select top method
best_method <- cv_comparison$method[1]

# 4. Evaluate on test set
test_metrics <- evaluate_test_metrics(results, X_test, y_test)

# 5. Visualize
plot_metrics(results, metric_type = "test")

# 6. Report final performance
test_metrics %>%
  filter(method == best_method) %>%
  select(method, f1_mean, f1_std)
```
:::




## Summary

GeneSelectR's metrics framework:

1. **Computes** standard classification metrics via scikit-learn
2. **Aggregates** scores across CV folds and test splits
3. **Visualizes** performance via customizable ggplot2 plots
4. **Guides** method selection through transparent reporting

The next chapter examines how to extract and interpret feature importances.

---

**Next**: [Feature Importance Estimation](07_feature_importance.qmd)
