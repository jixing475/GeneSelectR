---
title: "Core Classes: S4 Objects and Semantics"
keep-md: true
execute:
  cache: true
  freeze: auto
  warning: false
  message: false
---





## Why S4 for GeneSelectR?

R offers multiple object systems (S3, S4, R6, R7). GeneSelectR uses **S4** because:

1. **Formal contracts**: Slots have enforced types (e.g., `list`, `data.frame`)
2. **Method dispatch**: Generic functions can specialize on class signatures
3. **Bioconductor compatibility**: Most Bioconductor classes use S4 (e.g., `SummarizedExperiment`)
4. **Validation**: Custom validity methods ensure object consistency

The three core classes are:

- **`PipelineResults`**: Complete output from feature selection workflows
- **`GeneList`**: Gene identifiers in three naming systems
- **`AnnotatedGeneLists`**: Collection of `GeneList` objects per method

## Class Definitions

### `PipelineResults`

Container for all outputs from `perform_grid_search()`:




::: {.cell}

:::

::: {.cell}

```{.r .cell-code}
setClass("PipelineResults",
         slots = list(
           best_pipeline = "list",              # Winning hyperparameters
           cv_results = "list",                 # Full CV results per method
           inbuilt_feature_importance = "list", # Model-based importances
           permutation_importance = "list",     # Permutation importances
           cv_mean_score = "data.frame",        # Aggregated CV scores
           test_metrics = "TestMetrics"         # Test set performance (df or list)
         ))
```
:::




**Slot semantics**:

- `best_pipeline`: Named list with keys like `"method"`, `"scaler"`, `"params"`, `"score"`
- `cv_results`: Nested list where `cv_results[[method]]` contains sklearn's `cv_results_` dict
- `inbuilt_feature_importance`: List of numeric vectors (feature → importance)
- `permutation_importance`: List of data.frames with columns `feature`, `importance_mean`, `importance_std`
- `cv_mean_score`: data.frame with columns `method`, `mean_score`, `sd_score`, `rank`
- `test_metrics`: Either a data.frame (single split) or list of data.frames (multiple splits)

**Design rationale**: Keeping all outputs in one object simplifies downstream analysis and avoids scattered artifacts.

### Loading a Real Object

Let's load a `PipelineResults` fixture from the test suite:




::: {.cell}

```{.r .cell-code}
project_root <- "/Users/zero/Desktop/GeneSelectR"
devtools::load_all(project_root)
fixture_path <- file.path(project_root, "tests/testthat/fixtures/PipelineResults.rds")

pipeline_results <- readRDS(fixture_path)

# Inspect structure
cat("Class:", class(pipeline_results), "\n")
```

::: {.cell-output .cell-output-stdout}

```
Class: PipelineResults 
```


:::

```{.r .cell-code}
cat("Slots:", slotNames(pipeline_results), "\n")
```

::: {.cell-output .cell-output-stdout}

```
Slots: best_pipeline cv_results inbuilt_feature_importance permutation_importance cv_mean_score test_metrics 
```


:::
:::




#### Best Pipeline




::: {.cell}

```{.r .cell-code}
str(pipeline_results@best_pipeline, max.level = 1)
```
:::




The best pipeline includes the winning method name, hyperparameters, and CV score.

#### CV Mean Scores




::: {.cell}

```{.r .cell-code}
head(pipeline_results@cv_mean_score)
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




Each row represents a feature selection method, ranked by cross-validation performance.

#### Feature Importances




::: {.cell}

```{.r .cell-code}
# Inbuilt importances (top 5 features per method)
lapply(pipeline_results@inbuilt_feature_importance, function(x) head(x, 5))
```

::: {.cell-output .cell-output-stdout}

```
$Lasso
# A tibble: 5 × 8
  feature         mean_importance   std rank_Lasso_split_3 rank_Lasso_split_2
  <chr>                     <dbl> <dbl>              <int>              <int>
1 ENSG00000001630        0.00732     NA                  2                 NA
2 ENSG00000002726        0.000388    NA                 NA                  9
3 ENSG00000003436        0.00315     NA                 NA                  4
4 ENSG00000003756        0.000750    NA                  5                 NA
5 ENSG00000003989        0.000162    NA                 NA                 11
# ℹ 3 more variables: rank_Lasso_split_1 <int>, rank_Lasso_split_4 <int>,
#   rank_Lasso_split_5 <int>

$Univariate
# A tibble: 5 × 8
  feature mean_importance      std rank_Univariate_spli…¹ rank_Univariate_spli…²
  <chr>             <dbl>    <dbl>                  <int>                  <int>
1 ENSG00…         0.00718 NA                            1                     NA
2 ENSG00…         0.00745  0.00928                     NA                      1
3 ENSG00…         0.00163 NA                           NA                     NA
4 ENSG00…         0.00430 NA                            2                     NA
5 ENSG00…         0.00563 NA                           NA                     NA
# ℹ abbreviated names: ¹​rank_Univariate_split_2, ²​rank_Univariate_split_4
# ℹ 3 more variables: rank_Univariate_split_5 <int>,
#   rank_Univariate_split_3 <int>, rank_Univariate_split_1 <int>

$RandomForest
# A tibble: 5 × 8
  feature    mean_importance   std rank_RandomForest_sp…¹ rank_RandomForest_sp…²
  <chr>                <dbl> <dbl>                  <int>                  <int>
1 ENSG00000…        0.0133      NA                      1                     NA
2 ENSG00000…        0.00531     NA                     NA                      1
3 ENSG00000…        0.00260     NA                     NA                     NA
4 ENSG00000…        0.000733    NA                     NA                     NA
5 ENSG00000…        0.00306     NA                      4                     NA
# ℹ abbreviated names: ¹​rank_RandomForest_split_2, ²​rank_RandomForest_split_1
# ℹ 3 more variables: rank_RandomForest_split_3 <int>,
#   rank_RandomForest_split_5 <int>, rank_RandomForest_split_4 <int>

$boruta
# A tibble: 5 × 8
  feature          mean_importance   std rank_boruta_split_3 rank_boruta_split_2
  <chr>                      <dbl> <dbl>               <int>               <int>
1 ENSG00000003436         0.000369    NA                  57                  NA
2 ENSG00000003756         0.00205     NA                  14                  NA
3 ENSG00000004399         0.00132     NA                  18                  NA
4 ENSG00000004478…        0.000711    NA                  36                  NA
5 ENSG00000005001         0.000281    NA                  67                  NA
# ℹ 3 more variables: rank_boruta_split_4 <int>, rank_boruta_split_5 <int>,
#   rank_boruta_split_1 <int>
```


:::
:::




Each method yields a named numeric vector where names are feature identifiers and values are importance scores.

### `GeneList`

Simple container for gene identifiers in three formats:




::: {.cell}

```{.r .cell-code}
setClass("GeneList",
         slots = list(
           SYMBOL = "character",   # Official gene symbols (e.g., "TP53")
           ENSEMBL = "character",  # Ensembl IDs (e.g., "ENSG00000141510")
           ENTREZID = "character"  # NCBI Entrez IDs (e.g., "7157")
         ))
```
:::




**Why three formats?**

- `SYMBOL`: Human-readable, used in publications
- `ENSEMBL`: Stable across species, used in genomic databases
- `ENTREZID`: Required for clusterProfiler GO enrichment

Conversion is handled by `annotate_gene_lists()` using org.db annotation packages.

### `AnnotatedGeneLists`

Collection of `GeneList` objects, stratified by importance type:




::: {.cell}

```{.r .cell-code}
setClass("AnnotatedGeneLists",
         representation(
           inbuilt = "list",      # List of GeneList objects (model-based)
           permutation = "list"   # List of GeneList objects (permutation-based)
         ))
```
:::




**Structure**:

```
AnnotatedGeneLists
├── inbuilt
│   ├── Lasso → GeneList(SYMBOL, ENSEMBL, ENTREZID)
│   ├── RandomForest → GeneList(...)
│   └── Univariate → GeneList(...)
└── permutation
    ├── Lasso → GeneList(...)
    └── ...
```

Each named element corresponds to a feature selection method.

### Loading Annotated Gene Lists




::: {.cell}

```{.r .cell-code}
annotated_path <- file.path(project_root, "tests/testthat/fixtures/AnnotatedGeneLists.rds")
annotated_lists <- readRDS(annotated_path)

cat("Class:", class(annotated_lists), "\n")
```

::: {.cell-output .cell-output-stdout}

```
Class: AnnotatedGeneLists 
```


:::

```{.r .cell-code}
cat("Inbuilt methods:", names(annotated_lists@inbuilt), "\n")
```

::: {.cell-output .cell-output-stdout}

```
Inbuilt methods: Lasso Univariate RandomForest boruta background DEG_rural DEG_urban 
```


:::

```{.r .cell-code}
cat("Permutation methods:", names(annotated_lists@permutation), "\n")
```

::: {.cell-output .cell-output-stdout}

```
Permutation methods: Lasso Univariate RandomForest boruta background DEG_rural DEG_urban 
```


:::
:::




#### Inspecting a Single GeneList




::: {.cell}

```{.r .cell-code}
# Example: Lasso inbuilt features
lasso_genes <- annotated_lists@inbuilt$Lasso

cat("Number of genes:", length(lasso_genes@SYMBOL), "\n")
```

::: {.cell-output .cell-output-stdout}

```
Number of genes: 1118 
```


:::

```{.r .cell-code}
cat("First 5 symbols:", head(lasso_genes@SYMBOL, 5), "\n")
```

::: {.cell-output .cell-output-stdout}

```
First 5 symbols: CDK11A NADK ACOT7 DNAJC11 VAMP3 
```


:::

```{.r .cell-code}
cat("First 5 Ensembl:", head(lasso_genes@ENSEMBL, 5), "\n")
```

::: {.cell-output .cell-output-stdout}

```
First 5 Ensembl: ENSG00000008128 ENSG00000008130 ENSG00000097021 ENSG00000007923 ENSG00000049245 
```


:::
:::




## Class Usage Patterns

### Creating Objects Manually

Typically, users don't create these objects directly—they're returned by package functions. But for testing:




::: {.cell}

```{.r .cell-code}
my_genes <- new("GeneList",
                SYMBOL = c("TP53", "EGFR", "BRCA1"),
                ENSEMBL = c("ENSG00000141510", "ENSG00000146648", "ENSG00000012048"),
                ENTREZID = c("7157", "1956", "672"))
```
:::




### Accessing Slots




::: {.cell}

```{.r .cell-code}
# Direct slot access (discouraged in production code)
pipeline_results@best_pipeline

# Better: Define accessor methods
setGeneric("getBestPipeline", function(object) standardGeneric("getBestPipeline"))
setMethod("getBestPipeline", "PipelineResults", function(object) object@best_pipeline)
```
:::




GeneSelectR provides minimal accessors; users typically access slots directly in interactive analysis.

### Type Safety




::: {.cell}

```{.r .cell-code}
# This would fail at object creation:
bad_pipeline <- new("PipelineResults",
                    best_pipeline = "not a list",  # Type mismatch!
                    cv_results = list(),
                    ...)
# Error: invalid class "PipelineResults" object: 
# invalid object for slot "best_pipeline" in class "PipelineResults": 
# got class "character", should be or extend class "list"
```
:::




## Design Trade-offs

### S4 vs S3

**S4 advantages**:

- Compile-time type checking
- Formal documentation via `@slot` tags
- Method dispatch on multiple arguments (not used in GeneSelectR but available)

**S4 disadvantages**:

- Verbose syntax (`@` instead of `$`)
- Steeper learning curve
- Slower object creation (negligible for GeneSelectR's use case)

### S4 vs R6

R6 offers:

- Reference semantics (modify in-place)
- Private fields and methods
- Simpler inheritance

But GeneSelectR chose S4 for **immutability** and **Bioconductor alignment**. Pipeline results should not be modified after creation.

### Flat vs Nested Structures

`PipelineResults` could have been split into separate objects (`CVResults`, `ImportanceResults`, etc.), but:

- A single object simplifies API: `results <- perform_grid_search(...)`
- All related outputs are bundled for archiving (`saveRDS(results, "run1.rds")`)
- Downstream functions expect a consistent input type

## Summary

GeneSelectR's S4 classes provide:

1. **Type-safe containers** for complex ML outputs
2. **Self-documenting code** via slot definitions
3. **Seamless integration** with Bioconductor tools

The next chapter explores how these objects are populated via Python integration.

---

**Next**: [Python Integration: R ↔ Python Interface](04_python_integration.qmd)
