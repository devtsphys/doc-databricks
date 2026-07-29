# PySpark ML (MLlib) — Complete Reference Card

*DataFrame-based Machine Learning API (`pyspark.ml`) — architecture, full API tables, design patterns, and worked examples.*
*Covers Spark 3.x / 4.x. Last verified against Spark 4.1 (GA Spark Connect ML, nested-column support in feature transformers).*

---

## Table of Contents

1. [Architecture & Core Abstractions](#1-architecture--core-abstractions)
2. [Package Map](#2-package-map)
3. [Data Types: Vectors & Matrices](#3-data-types-vectors--matrices)
4. [Feature Engineering (`pyspark.ml.feature`)](#4-feature-engineering-pysparkmlfeature)
5. [Classification (`pyspark.ml.classification`)](#5-classification-pysparkmlclassification)
6. [Regression (`pyspark.ml.regression`)](#6-regression-pysparkmlregression)
7. [Clustering (`pyspark.ml.clustering`)](#7-clustering-pysparkmlclustering)
8. [Recommendation (`pyspark.ml.recommendation`)](#8-recommendation-pysparkmlrecommendation)
9. [Frequent Pattern Mining (`pyspark.ml.fpm`)](#9-frequent-pattern-mining-pysparkmlfpm)
10. [Evaluation (`pyspark.ml.evaluation`)](#10-evaluation-pysparkmlevaluation)
11. [Model Selection & Tuning (`pyspark.ml.tuning`)](#11-model-selection--tuning-pysparkmltuning)
12. [Pipelines in Depth](#12-pipelines-in-depth)
13. [Statistics (`pyspark.ml.stat`)](#13-statistics-pysparkmlstat)
14. [Persistence, MLflow & Unity Catalog](#14-persistence-mlflow--unity-catalog)
15. [Distributed Computing & Performance](#15-distributed-computing--performance)
16. [Design Patterns & Best Practices](#16-design-patterns--best-practices)
17. [Common Pitfalls](#17-common-pitfalls)
18. [Appendix: Full Class Reference Table](#18-appendix-full-class-reference-table)
19. [End-to-End Worked Example](#19-end-to-end-worked-example)

---

## 1. Architecture & Core Abstractions

`pyspark.ml` is the **DataFrame-based** ML API and is the actively developed, recommended API. `pyspark.mllib` is the legacy **RDD-based** API — in maintenance mode since Spark 2.0, receives no new features, and is not covered here.

| Concept | Definition | Key Method |
|---|---|---|
| **Transformer** | An algorithm that maps one DataFrame to another (feature transform, or a fitted model applying predictions). Stateless w.r.t. the call itself. | `transform(df) -> DataFrame` |
| **Estimator** | An algorithm that can be *fit* on a DataFrame to produce a Transformer (a Model). Encapsulates a learning algorithm or fitting procedure. | `fit(df) -> Model` |
| **Model** | The output of `Estimator.fit()`. It **is** a Transformer (e.g. `LogisticRegressionModel`). | `transform(df)` |
| **Pipeline** | Chains multiple Transformers/Estimators into a single workflow; itself an Estimator. | `fit(df) -> PipelineModel` |
| **PipelineModel** | The fitted Pipeline; itself a Transformer. | `transform(df)` |
| **Param** | Self-contained, named, typed hyperparameter with a uniform get/set API across all algorithms. | `getParam`, `setParam`, `explainParams()` |
| **ParamMap** | A set of `(Param, value)` pairs, used to override defaults, e.g. in grid search. | — |

**Uniform Param API** (works the same on every Estimator/Transformer):

```python
lr = LogisticRegression(maxIter=10, regParam=0.01)
lr.getMaxIter()                 # 10
lr.setRegParam(0.1)             # fluent setter, returns self
lr.explainParams()              # human-readable listing of all params + defaults
lr.extractParamMap()            # dict of all set params
```

**Estimator/Transformer/Pipeline relationship:**

```
Estimator.fit(df)        -> Model (a Transformer)
Transformer.transform(df)-> DataFrame
Pipeline([stage1, stage2, ..., stageN]).fit(df) -> PipelineModel
PipelineModel.transform(df) -> DataFrame  (runs all stages in order)
```

All `pyspark.ml` algorithms operate on **DataFrames with a vector-typed feature column** (conventionally named `features`, of type `VectorUDT`), produced via `VectorAssembler`. This is the single most important structural convention in the whole API.

---

## 2. Package Map

| Module | Contents |
|---|---|
| `pyspark.ml.feature` | Feature extractors, transformers, selectors |
| `pyspark.ml.classification` | Classification algorithms |
| `pyspark.ml.regression` | Regression algorithms |
| `pyspark.ml.clustering` | Clustering algorithms |
| `pyspark.ml.recommendation` | ALS collaborative filtering |
| `pyspark.ml.fpm` | Frequent pattern / association rule mining |
| `pyspark.ml.evaluation` | Metric evaluators |
| `pyspark.ml.tuning` | `ParamGridBuilder`, `CrossValidator`, `TrainValidationSplit` |
| `pyspark.ml.pipeline` | `Pipeline`, `PipelineModel` |
| `pyspark.ml.param` / `pyspark.ml.param.shared` | Param infrastructure, shared mixins (`HasMaxIter`, `HasRegParam`, …) |
| `pyspark.ml.linalg` | `Vector`, `DenseVector`, `SparseVector`, `Matrix`, `Vectors`, `Matrices` |
| `pyspark.ml.stat` | `Correlation`, `ChiSquareTest`, `Summarizer`, `KolmogorovSmirnovTest`, `ANOVATest`, `FValueTest` |
| `pyspark.ml.util` | `MLReader`/`MLWriter`, `MLReadable`/`MLWritable`, `Identifiable`, `DefaultParamsReadable` |
| `pyspark.ml.functions` | `vector_to_array`, `array_to_vector`, `predict_batch_udf` |
| `pyspark.ml.torch.distributor` | `TorchDistributor` — distributed PyTorch training launcher |
| `pyspark.ml.connect` | Spark Connect–native ML client (GA for Python in Spark 4.1) |

---

## 3. Data Types: Vectors & Matrices

`pyspark.ml.linalg` (note: **not** interchangeable with `pyspark.mllib.linalg` — different classes, must be converted explicitly).

| Class | Description | Example |
|---|---|---|
| `Vectors.dense(*values)` | Dense vector factory | `Vectors.dense([1.0, 0.0, 3.0])` |
| `Vectors.sparse(size, indices, values)` | Sparse vector factory | `Vectors.sparse(4, [0, 2], [1.0, 3.0])` |
| `DenseVector` | Dense vector class; `.toArray()` → numpy array | `dv.toArray()` |
| `SparseVector` | Sparse vector class; `.indices`, `.values`, `.toArray()` | `sv.indices` |
| `VectorUDT` | The Spark SQL column type backing vector columns | schema introspection |
| `Matrices.dense(rows, cols, values)` | Dense matrix factory | — |
| `DenseMatrix` / `SparseMatrix` | Matrix classes (mostly used internally / model coefficients) | `model.coefficientMatrix` |

```python
from pyspark.ml.linalg import Vectors, DenseVector

# A DataFrame column of type vector is required by every ML algorithm
df = spark.createDataFrame([
    (1, Vectors.dense([0.0, 1.1, 0.1])),
    (0, Vectors.sparse(3, [1], [1.0])),
], ["label", "features"])
```

Conversion helpers (`pyspark.ml.functions`, Spark 3.1+) let you move between SQL `array<double>` columns and ML `vector` columns without UDFs (Catalyst-optimized):

```python
from pyspark.ml.functions import vector_to_array, array_to_vector

df.select(vector_to_array("features").alias("arr"))
df.select(array_to_vector("arr").alias("features"))
```

---

## 4. Feature Engineering (`pyspark.ml.feature`)

All entries are **Transformers** unless marked **Estimator** (Estimators must be `.fit()` before use, producing a `*Model` Transformer).

| Class | Type | Purpose | Key Params |
|---|---|---|---|
| `Binarizer` | Transformer | Threshold numeric column into 0/1 | `threshold`, `inputCol`, `outputCol` |
| `Bucketizer` | Transformer | Bin continuous values into fixed buckets | `splits`, `handleInvalid` |
| `QuantileDiscretizer` | Estimator | Bin into `numBuckets` equal-frequency bins (approx quantiles) | `numBuckets`, `relativeError` |
| `StandardScaler` | Estimator | Zero mean / unit variance scaling | `withMean`, `withStd` |
| `MinMaxScaler` | Estimator | Rescale to `[min, max]` range | `min`, `max` |
| `MaxAbsScaler` | Estimator | Scale by max absolute value (preserves sparsity) | — |
| `RobustScaler` | Estimator | Scale using median/IQR (outlier-robust) | `lower`, `upper`, `withCentering` |
| `Normalizer` | Transformer | Scale each **row** to unit p-norm | `p` (default 2.0) |
| `PCA` | Estimator | Principal component projection | `k` |
| `StringIndexer` | Estimator | Encode string labels/categories to numeric indices by frequency | `handleInvalid` (`error`/`skip`/`keep`), `stringOrderType` |
| `IndexToString` | Transformer | Inverse of `StringIndexer` | `labels` |
| `OneHotEncoder` | Estimator | One-hot encode numeric category indices | `dropLast`, `handleInvalid` |
| `VectorIndexer` | Estimator | Auto-detect + index categorical columns inside a vector | `maxCategories` |
| `VectorAssembler` | Transformer | Combine multiple columns into a single `features` vector | `inputCols`, `outputCol`, `handleInvalid` |
| `VectorSlicer` | Transformer | Select a subset of vector indices/names | `indices`, `names` |
| `VectorSizeHint` | Transformer | Attach expected vector size (helps Catalyst validate/optimize) | `size` |
| `ElementwiseProduct` | Transformer | Hadamard product with a scaling vector | `scalingVec` |
| `Interaction` | Transformer | Pairwise feature interactions | `inputCols` |
| `PolynomialExpansion` | Transformer | Expand into polynomial feature space | `degree` |
| `Imputer` | Estimator | Fill missing values with mean/median/mode | `strategy`, `missingValue` |
| `Tokenizer` | Transformer | Lowercase whitespace tokenizer | — |
| `RegexTokenizer` | Transformer | Regex-based tokenizer | `pattern`, `gaps` |
| `StopWordsRemover` | Transformer | Remove stop words from token arrays | `stopWords`, `locale` |
| `NGram` | Transformer | Build n-grams from token arrays | `n` |
| `HashingTF` | Transformer | Term frequency via feature hashing | `numFeatures` |
| `CountVectorizer` | Estimator | Vocabulary-based term frequency (invertible, unlike hashing) | `vocabSize`, `minDF` |
| `IDF` | Estimator | Inverse document frequency weighting (pairs with `HashingTF`/`CountVectorizer`) | `minDocFreq` |
| `Word2Vec` | Estimator | Dense word/document embeddings | `vectorSize`, `windowSize` |
| `FeatureHasher` | Transformer | Hash heterogeneous columns (numeric+categorical) directly into one vector | `numFeatures`, `categoricalCols` |
| `ChiSqSelector` | Estimator | Select top features by chi-squared test (categorical label) | `numTopFeatures`, `selectorType` |
| `UnivariateFeatureSelector` | Estimator | Generalized selector (chi2 / ANOVA / F-value, auto by label/feature type) | `featureType`, `labelType`, `selectionMode` |
| `VarianceThresholdSelector` | Estimator | Drop low-variance features | `varianceThreshold` |
| `DCT` | Transformer | Discrete Cosine Transform | `inverse` |
| `RFormula` | Estimator | R-style formula (`"y ~ a + b"`) → auto-assembles `features`/`label` | `formula` |
| `SQLTransformer` | Transformer | Arbitrary SQL as a pipeline stage (`__THIS__` refers to input df) | `statement` |
| `BucketedRandomProjectionLSH` | Estimator | LSH for Euclidean distance ANN | `bucketLength`, `numHashTables` |
| `MinHashLSH` | Estimator | LSH for Jaccard distance ANN (sets) | `numHashTables` |

**Typical feature pipeline snippet:**

```python
from pyspark.ml.feature import StringIndexer, OneHotEncoder, VectorAssembler, StandardScaler

indexer  = StringIndexer(inputCol="city", outputCol="city_idx", handleInvalid="keep")
encoder  = OneHotEncoder(inputCols=["city_idx"], outputCols=["city_ohe"])
assembler = VectorAssembler(inputCols=["age", "income", "city_ohe"], outputCol="features_raw")
scaler   = StandardScaler(inputCol="features_raw", outputCol="features")
```

---

## 5. Classification (`pyspark.ml.classification`)

All produce a `*Model` on `.fit()`. Most support `predictionCol`, `probabilityCol`, `rawPredictionCol` outputs.

| Class | Type | Handles | Key Params | Notes |
|---|---|---|---|---|
| `LogisticRegression` | Binary + Multinomial | numeric/binary label | `maxIter`, `regParam`, `elasticNetParam`, `family` | `family="multinomial"` for >2 classes; `elasticNetParam` blends L1/L2 |
| `DecisionTreeClassifier` | Multiclass | categorical/numeric | `maxDepth`, `maxBins`, `impurity` (`gini`/`entropy`) | Base learner for RF/GBT |
| `RandomForestClassifier` | Multiclass | — | `numTrees`, `maxDepth`, `featureSubsetStrategy` | Bagging ensemble |
| `GBTClassifier` | Binary only | — | `maxIter`, `stepSize`, `lossType` | Gradient-boosted trees; binary label only |
| `NaiveBayes` | Multiclass | non-negative features (or Bernoulli/Gaussian) | `modelType` (`multinomial`/`bernoulli`/`gaussian`), `smoothing` | Fast, good text baseline |
| `LinearSVC` | Binary only | — | `maxIter`, `regParam` | Linear SVM, hinge loss |
| `MultilayerPerceptronClassifier` | Multiclass | — | `layers`, `maxIter`, `blockSize` | Feed-forward neural net |
| `OneVsRest` | Meta-estimator | wraps any binary classifier | `classifier` | Turns any binary classifier into multiclass |
| `FMClassifier` | Binary | — | `factorSize`, `maxIter` | Factorization Machines — good for sparse categorical interactions |

```python
from pyspark.ml.classification import LogisticRegression

lr = LogisticRegression(featuresCol="features", labelCol="label",
                         maxIter=50, regParam=0.01, elasticNetParam=0.5)
model = lr.fit(train_df)
preds = model.transform(test_df)          # adds prediction, probability, rawPrediction
model.coefficients, model.intercept
model.summary.areaUnderROC                # training summary (where available)
```

---

## 6. Regression (`pyspark.ml.regression`)

| Class | Purpose | Key Params | Notes |
|---|---|---|---|
| `LinearRegression` | OLS / regularized linear regression | `regParam`, `elasticNetParam`, `loss` (`squaredError`/`huber`) | `summary` gives R², RMSE, coefficient std errors |
| `GeneralizedLinearRegression` | GLM with configurable family/link | `family` (`gaussian`/`binomial`/`poisson`/`gamma`/`tweedie`), `link` | Only single-machine-scale (< 4096 features) |
| `DecisionTreeRegressor` | Tree regression | `maxDepth`, `maxBins` | — |
| `RandomForestRegressor` | Ensemble | `numTrees`, `maxDepth` | — |
| `GBTRegressor` | Gradient boosting | `maxIter`, `stepSize`, `lossType` (`squared`/`absolute`) | — |
| `AFTSurvivalRegression` | Survival analysis (accelerated failure time) | `censorCol`, `quantileProbabilities` | Time-to-event modeling |
| `IsotonicRegression` | Monotonic (non-decreasing/increasing) fit | `isotonic`, `featureIndex` | Calibration use-case |
| `FMRegressor` | Factorization Machines regression | `factorSize` | Sparse interaction-heavy features |

```python
from pyspark.ml.regression import LinearRegression

lr = LinearRegression(featuresCol="features", labelCol="label", regParam=0.1)
model = lr.fit(train_df)
print(model.summary.rootMeanSquaredError, model.summary.r2)
```

---

## 7. Clustering (`pyspark.ml.clustering`)

| Class | Algorithm | Key Params | Notes |
|---|---|---|---|
| `KMeans` | Lloyd's k-means / k-means\|\| init | `k`, `initMode`, `maxIter`, `distanceMeasure` (`euclidean`/`cosine`) | Most common baseline |
| `BisectingKMeans` | Hierarchical divisive k-means | `k`, `maxIter` | Often faster & more stable than flat k-means |
| `GaussianMixture` | Soft clustering via EM | `k`, `maxIter` | Outputs cluster probabilities |
| `LDA` | Latent Dirichlet Allocation (topic modeling) | `k`, `maxIter`, `optimizer` (`online`/`em`) | Text/topic modeling |
| `PowerIterationClustering` | Graph-based clustering on similarity matrix | `k`, `maxIter` | Transform-only (no `fit`), takes `(src, dst, weight)` edges |

```python
from pyspark.ml.clustering import KMeans

km = KMeans(featuresCol="features", k=5, seed=42)
model = km.fit(df)
model.clusterCenters()
model.summary.trainingCost      # within-set sum of squared errors
```

---

## 8. Recommendation (`pyspark.ml.recommendation`)

| Class | Purpose | Key Params |
|---|---|---|
| `ALS` | Alternating Least Squares matrix factorization for collaborative filtering | `rank`, `maxIter`, `regParam`, `implicitPrefs`, `coldStartStrategy`, `nonnegative` |

```python
from pyspark.ml.recommendation import ALS

als = ALS(userCol="userId", itemCol="itemId", ratingCol="rating",
          rank=10, maxIter=10, regParam=0.1,
          implicitPrefs=False, coldStartStrategy="drop")   # "drop" avoids NaN metrics on unseen users/items
model = als.fit(train_df)
model.recommendForAllUsers(10)
model.recommendForAllItems(10)
```

Set `implicitPrefs=True` for click/watch/purchase-count data (uses a confidence-weighted loss rather than treating counts as explicit ratings).

---

## 9. Frequent Pattern Mining (`pyspark.ml.fpm`)

| Class | Purpose | Key Params |
|---|---|---|
| `FPGrowth` | Frequent itemsets + association rules | `minSupport`, `minConfidence`, `itemsCol` |
| `PrefixSpan` | Sequential pattern mining | `minSupport`, `maxPatternLength` |

```python
from pyspark.ml.fpm import FPGrowth

fp = FPGrowth(itemsCol="items", minSupport=0.1, minConfidence=0.6)
model = fp.fit(df)                # df: one row per basket, "items": array<string>
model.freqItemsets.show()
model.associationRules.show()     # antecedent, consequent, confidence, lift
```

---

## 10. Evaluation (`pyspark.ml.evaluation`)

Every evaluator implements `.evaluate(dataframe)` and `.isLargerBetter()`.

| Class | Task | `metricName` options |
|---|---|---|
| `BinaryClassificationEvaluator` | Binary classification | `areaUnderROC`, `areaUnderPR` |
| `MulticlassClassificationEvaluator` | Multiclass classification | `accuracy`, `f1`, `weightedPrecision`, `weightedRecall`, `weightedFMeasure`, `logLoss`, `hammingLoss` |
| `MultilabelClassificationEvaluator` | Multilabel classification | `subsetAccuracy`, `f1Measure`, `precision`, `recall`, `hammingLoss` |
| `RegressionEvaluator` | Regression | `rmse`, `mse`, `mae`, `r2`, `var` |
| `ClusteringEvaluator` | Clustering | `silhouette` (squared euclidean or cosine distance) |
| `RankingEvaluator` | Ranking (Spark 3.0+) | `meanAveragePrecision`, `ndcgAtK`, `precisionAtK` |

```python
from pyspark.ml.evaluation import BinaryClassificationEvaluator, MulticlassClassificationEvaluator

bce = BinaryClassificationEvaluator(labelCol="label", rawPredictionCol="rawPrediction",
                                     metricName="areaUnderROC")
auc = bce.evaluate(predictions)

mce = MulticlassClassificationEvaluator(labelCol="label", predictionCol="prediction",
                                         metricName="f1")
f1 = mce.evaluate(predictions)
```

---

## 11. Model Selection & Tuning (`pyspark.ml.tuning`)

| Class | Purpose | Key Params |
|---|---|---|
| `ParamGridBuilder` | Fluent builder for hyperparameter grids | `.addGrid(param, values)`, `.build()` |
| `CrossValidator` | k-fold cross-validation over a grid | `estimator`, `estimatorParamMaps`, `evaluator`, `numFolds`, `parallelism`, `seed` |
| `CrossValidatorModel` | Fitted CV result | `.bestModel`, `.avgMetrics` |
| `TrainValidationSplit` | Single train/validation split (cheaper than k-fold) | `trainRatio`, `parallelism` |
| `TrainValidationSplitModel` | Fitted TVS result | `.bestModel`, `.validationMetrics` |

```python
from pyspark.ml.tuning import ParamGridBuilder, CrossValidator
from pyspark.ml.classification import RandomForestClassifier
from pyspark.ml.evaluation import BinaryClassificationEvaluator

rf = RandomForestClassifier(featuresCol="features", labelCol="label")

grid = (ParamGridBuilder()
        .addGrid(rf.numTrees, [50, 100, 200])
        .addGrid(rf.maxDepth, [5, 10, 15])
        .build())

cv = CrossValidator(estimator=rf,
                     estimatorParamMaps=grid,
                     evaluator=BinaryClassificationEvaluator(),
                     numFolds=5,
                     parallelism=4,       # fit folds concurrently — big wall-clock win on a cluster
                     seed=42)

cv_model = cv.fit(train_df)
best_rf = cv_model.bestModel
cv_model.avgMetrics                       # list of avg metric per grid point
```

`CrossValidator`/`TrainValidationSplit` also accept a full **`Pipeline`** as the `estimator` — this refits every stage (including feature transformers) inside each fold, which is the statistically correct way to tune when any upstream stage is itself fit on data (e.g. `StandardScaler`, `StringIndexer`).

---

## 12. Pipelines in Depth

```python
from pyspark.ml import Pipeline

pipeline = Pipeline(stages=[indexer, encoder, assembler, scaler, classifier])
model = pipeline.fit(train_df)            # -> PipelineModel
predictions = model.transform(test_df)

model.stages                              # list of fitted stage objects, in order
model.stages[-1]                          # the fitted final estimator (e.g. LogisticRegressionModel)
```

**Key mechanics:**
- A `Pipeline` stage list may mix Transformers (no fitting needed) and Estimators (fit in sequence, each stage seeing the output of the previous).
- Nested pipelines are supported: a `Pipeline` can itself be a stage inside another `Pipeline`.
- `PipelineModel.write().overwrite().save(path)` / `PipelineModel.load(path)` persists the **entire fitted DAG**, including learned vocabulary, scaler statistics, string-index mappings, and model coefficients — this is what makes pipelines the unit of production deployment, not the raw model.
- Params can be inspected/overridden per-stage via `pipeline.getStages()[i].extractParamMap()`.

---

## 13. Statistics (`pyspark.ml.stat`)

| Class / Function | Purpose |
|---|---|
| `Correlation.corr(df, column, method)` | Pairwise correlation matrix (`pearson`/`spearman`) over a vector column |
| `ChiSquareTest.test(df, featuresCol, labelCol)` | Chi-squared independence test per feature vs. categorical label |
| `Summarizer` | Vector column summary stats (`mean`, `variance`, `count`, `numNonZeros`, `max`, `min`, `normL1`, `normL2`) |
| `KolmogorovSmirnovTest.test(df, sampleCol, distName, *params)` | Goodness-of-fit test vs. a theoretical distribution |
| `ANOVATest.test(df, featuresCol, labelCol)` | ANOVA F-test, continuous features vs. categorical label |
| `FValueTest.test(df, featuresCol, labelCol)` | F-test, continuous features vs. continuous label |

```python
from pyspark.ml.stat import Correlation, ChiSquareTest, Summarizer

corr_matrix = Correlation.corr(df, "features", "pearson").head()[0]

chi = ChiSquareTest.test(df, "features", "label").head()
chi.pValues, chi.statistics, chi.degreesOfFreedom

df.select(Summarizer.metrics("mean", "variance", "count").summary("features")).show(truncate=False)
```

---

## 14. Persistence, MLflow & Unity Catalog

**Native save/load** (every Estimator, Model, Transformer, and Pipeline implements `MLWritable`/`MLReadable`):

```python
model.write().overwrite().save("/path/to/model")
from pyspark.ml.classification import LogisticRegressionModel
loaded = LogisticRegressionModel.load("/path/to/model")

pipeline_model.write().overwrite().save("/path/to/pipeline_model")
```

**MLflow integration** (the standard pattern on Databricks):

```python
import mlflow
import mlflow.spark

mlflow.pyspark.ml.autolog()               # auto-logs params, metrics, and the fitted pipeline per run

with mlflow.start_run(run_name="rf_churn_model"):
    model = pipeline.fit(train_df)
    preds = model.transform(test_df)
    auc = evaluator.evaluate(preds)

    mlflow.log_metric("auc", auc)
    mlflow.spark.log_model(model, artifact_path="model",
                            registered_model_name="catalog.schema.churn_model")   # Unity Catalog 3-level name
```

- `mlflow.spark.log_model` persists a `PipelineModel` as a single deployable artifact (feature engineering + model together) — avoids train/serve skew from re-implementing preprocessing at inference time.
- Register to the **Unity Catalog Model Registry** using a `catalog.schema.model_name` path for governed, environment-portable versioning (aligns with lineage/access control already applied to the underlying Delta tables).
- For batch scoring, load back with `mlflow.spark.load_model` (returns a `PipelineModel`) or wrap as a `pyspark_udf` via `mlflow.pyfunc.spark_udf` for row-wise scoring inside a larger DataFrame pipeline.
- Spark 4.1 brought **Spark Connect ML to GA for Python clients**, including smarter model caching — `pyspark.ml` workloads (including `mlflow.spark`) now run on **serverless compute** in Databricks (environment version 4+), with hyperparameter search there recommended via **Optuna + Joblib Spark** rather than `CrossValidator`'s built-in parallelism.

---

## 15. Distributed Computing & Performance

- **Cache/persist before iterative algorithms.** ALS, LDA, k-means, and tree ensembles scan the training DataFrame repeatedly across iterations — `df.cache()` (or `.persist(StorageLevel.MEMORY_AND_DISK)`) before `.fit()` avoids recomputing upstream stages each pass.
- **`maxBins` trade-off for trees.** Higher `maxBins` gives finer split candidates (better splits) at higher shuffle/compute cost; must be ≥ the number of categories in any single categorical feature.
- **`featureSubsetStrategy` for forests.** Controls how many features are considered per split (`"auto"`, `"sqrt"`, `"log2"`, a fraction, or an integer) — the main lever for RF training speed vs. quality.
- **Avoid UDFs on vector columns.** Use `pyspark.ml.functions.vector_to_array`/`array_to_vector` and built-in transformers instead of Python UDFs touching `Vector` objects — UDFs break Catalyst/Tungsten optimization and serialize per-row through Python.
- **`CrossValidator(parallelism=N)`.** Fits folds/grid points concurrently on the driver's thread pool — set this to your available parallel task slots; each fold still runs a full distributed Spark job.
- **Broadcast small models/lookup tables.** When scoring with a small fitted model object outside the native `transform()` path (e.g., inside a `predict_batch_udf`), broadcast it rather than re-shipping per task.
- **Checkpointing for long iterative chains.** `ALS`/`LDA`/tree ensembles with many iterations can build long lineage graphs; set `sc.setCheckpointDir(...)` and use `checkpointInterval` to truncate lineage and avoid stack/recomputation blowups.
- **Repartition to avoid skew.** Especially relevant for `ALS` on implicit-feedback data with power-law user/item activity — pre-repartition or salt heavy keys before fitting.
- **Prefer built-in feature transformers over Pandas UDFs where possible**; when Pandas UDFs/Pandas API on Spark are needed for complex feature logic, keep them as an isolated upstream stage, not intermixed row-by-row with `pyspark.ml` transformers.
- **`GBTClassifier` is binary-only** — for multiclass boosting use `OneVsRest(classifier=GBTClassifier(...))` or switch to `RandomForestClassifier`.

---

## 16. Design Patterns & Best Practices

**Fit-on-train-only.** Every `Estimator` in the feature stage (`StringIndexer`, `OneHotEncoder`, scalers, `Imputer`, `CountVectorizer`) must be `.fit()` on the **training split only**, then applied via `.transform()` to validation/test/production data. Wrapping these inside a `Pipeline` and calling `pipeline.fit(train_df)` enforces this automatically — it's the main reason to prefer pipelines over manually chained transforms.

**Handle unseen categories defensively.** Set `handleInvalid="keep"` (adds an "unknown" bucket) on `StringIndexer`/`VectorAssembler` for anything that will see production traffic with categories not present at training time; `"error"` (the default on some transformers) will crash the serving job on the first unseen value.

**Tune the whole Pipeline, not just the model.** Pass the full `Pipeline` (not just the final estimator) as `CrossValidator`'s `estimator` whenever any upstream stage is data-dependent (scalers, indexers) — otherwise cross-validation folds "leak" through a single global fit of those stages.

**Feature Store pattern (Databricks).** Compute features once (e.g. with the Databricks Feature Engineering client) and look them up by primary key at both training and serving time, rather than re-deriving feature logic separately in the training pipeline and the serving path — this is the standard way to eliminate train/serve skew, and pairs naturally with `pyspark.ml.Pipeline` as the modeling layer on top of the assembled feature table.

**Persist Pipelines, not bare models.** Log/register the fitted `PipelineModel` (feature engineering + model) as one artifact via `mlflow.spark.log_model`, so a scoring job only needs raw input columns, not hand-reimplemented preprocessing.

**Bound categorical cardinality for high-cardinality columns.** Prefer `FeatureHasher`/`HashingTF` over `OneHotEncoder` when a categorical column has unbounded or very high cardinality (e.g. free-text IDs) — `OneHotEncoder` on such a column blows up the feature-vector width and downstream memory.

**Set seeds everywhere.** `KMeans`, `RandomForestClassifier`, train/test `randomSplit`, `CrossValidator`, etc. all take a `seed` — set it explicitly for reproducible experiments.

**Prefer `RFormula` for quick baselines / parity with R workflows**, but move to explicit `StringIndexer`/`OneHotEncoder`/`VectorAssembler` stages for production pipelines — `RFormula` is convenient but less transparent/controllable stage-by-stage.

**Log model signature + input example with MLflow** so downstream `mlflow.pyfunc` serving/batch-inference validates schema automatically rather than failing at inference time with an obscure Spark error.

**Use `parallelism` in `CrossValidator`/`TrainValidationSplit`** to fit multiple folds/grid points concurrently — this is a cheap, easy win when the driver isn't already saturated by other work.

---

## 17. Common Pitfalls

| Pitfall | Consequence | Fix |
|---|---|---|
| Fitting `StringIndexer`/`StandardScaler`/`Imputer` on the full dataset before splitting | Train/test leakage, inflated validation metrics | Fit only on train (wrap in a `Pipeline`, fit on `train_df`) |
| `StringIndexer` default `handleInvalid="error"` in production | Job crashes on first unseen category | Set `handleInvalid="keep"` |
| Passing `pyspark.mllib.linalg.Vector` where `pyspark.ml.linalg.Vector` is expected (or vice versa) | `TypeError` / silent schema mismatch | Never mix the two linalg packages; `pyspark.ml` only |
| Treating a `Vector` column value as a numpy array directly | `AttributeError` | Call `.toArray()` first |
| `VectorAssembler` input column containing nulls | Raises an error (or silently drops rows depending on `handleInvalid`) | `Imputer` upstream, or set `handleInvalid="skip"/"keep"` deliberately |
| Using `GBTClassifier` for a >2-class problem | Raises an error — GBT classifier is binary only | Use `RandomForestClassifier` or wrap in `OneVsRest` |
| Comparing `CrossValidator.avgMetrics` across differently-shaped grids without matching evaluator direction | Silently picks a "best" model that isn't actually best (e.g. for a lower-is-better metric) | Check `evaluator.isLargerBetter()`; `CrossValidator` handles this correctly internally, but manual comparisons must too |
| Not caching before `ALS`/`LDA`/tree ensemble `.fit()` | Repeated recomputation of upstream DataFrame lineage each iteration, huge slowdown | `.cache()`/`.persist()` the training DataFrame first |
| Using UDFs to manipulate `Vector` columns row-by-row | Kills Catalyst/Tungsten optimizations, slow | Use `vector_to_array`/`array_to_vector` + built-in SQL functions where possible |
| Forgetting `coldStartStrategy="drop"` on `ALS` | `NaN` predictions for unseen users/items break evaluation metrics | Set `coldStartStrategy="drop"` before evaluating |
| Assuming `OneHotEncoder` needs string input directly | It expects **numeric indices** (output of `StringIndexer`), not raw strings | `StringIndexer` → `OneHotEncoder`, always in that order |

---

## 18. Appendix: Full Class Reference Table

Consolidated index of every class covered above — module, category, and role, for quick lookup.

| Class | Module | Category | Role |
|---|---|---|---|
| `Pipeline` / `PipelineModel` | `pyspark.ml` | Core | Chain of stages |
| `Transformer` / `Estimator` / `Model` | `pyspark.ml` | Core | Base abstractions |
| `Param` / `Params` / `ParamMap` | `pyspark.ml.param` | Core | Hyperparameter infra |
| `Vectors` / `DenseVector` / `SparseVector` / `VectorUDT` | `pyspark.ml.linalg` | Data type | Feature vector representation |
| `Matrices` / `DenseMatrix` / `SparseMatrix` | `pyspark.ml.linalg` | Data type | Matrix representation |
| `Binarizer` | `pyspark.ml.feature` | Feature — Transformer | Threshold to 0/1 |
| `Bucketizer` | `pyspark.ml.feature` | Feature — Transformer | Fixed-split binning |
| `QuantileDiscretizer` | `pyspark.ml.feature` | Feature — Estimator | Equal-frequency binning |
| `StandardScaler` | `pyspark.ml.feature` | Feature — Estimator | Zero-mean/unit-variance scaling |
| `MinMaxScaler` | `pyspark.ml.feature` | Feature — Estimator | Range rescaling |
| `MaxAbsScaler` | `pyspark.ml.feature` | Feature — Estimator | Sparsity-preserving scaling |
| `RobustScaler` | `pyspark.ml.feature` | Feature — Estimator | Median/IQR scaling |
| `Normalizer` | `pyspark.ml.feature` | Feature — Transformer | Row-wise p-norm scaling |
| `PCA` | `pyspark.ml.feature` | Feature — Estimator | Dimensionality reduction |
| `StringIndexer` / `StringIndexerModel` | `pyspark.ml.feature` | Feature — Estimator | Category → index |
| `IndexToString` | `pyspark.ml.feature` | Feature — Transformer | Index → category |
| `OneHotEncoder` | `pyspark.ml.feature` | Feature — Estimator | Index → one-hot |
| `VectorIndexer` | `pyspark.ml.feature` | Feature — Estimator | Auto categorical detection in vectors |
| `VectorAssembler` | `pyspark.ml.feature` | Feature — Transformer | Columns → single vector |
| `VectorSlicer` | `pyspark.ml.feature` | Feature — Transformer | Vector subsetting |
| `VectorSizeHint` | `pyspark.ml.feature` | Feature — Transformer | Vector size metadata |
| `ElementwiseProduct` | `pyspark.ml.feature` | Feature — Transformer | Hadamard scaling |
| `Interaction` | `pyspark.ml.feature` | Feature — Transformer | Pairwise interactions |
| `PolynomialExpansion` | `pyspark.ml.feature` | Feature — Transformer | Polynomial features |
| `Imputer` / `ImputerModel` | `pyspark.ml.feature` | Feature — Estimator | Missing value fill |
| `Tokenizer` / `RegexTokenizer` | `pyspark.ml.feature` | Feature — Transformer | Text → tokens |
| `StopWordsRemover` | `pyspark.ml.feature` | Feature — Transformer | Remove stop words |
| `NGram` | `pyspark.ml.feature` | Feature — Transformer | Token n-grams |
| `HashingTF` | `pyspark.ml.feature` | Feature — Transformer | Hashed term frequency |
| `CountVectorizer` / `CountVectorizerModel` | `pyspark.ml.feature` | Feature — Estimator | Vocabulary term frequency |
| `IDF` / `IDFModel` | `pyspark.ml.feature` | Feature — Estimator | Inverse doc frequency |
| `Word2Vec` / `Word2VecModel` | `pyspark.ml.feature` | Feature — Estimator | Word embeddings |
| `FeatureHasher` | `pyspark.ml.feature` | Feature — Transformer | Mixed-type hashing |
| `ChiSqSelector` | `pyspark.ml.feature` | Feature — Estimator | Chi² feature selection |
| `UnivariateFeatureSelector` | `pyspark.ml.feature` | Feature — Estimator | Generalized statistical selection |
| `VarianceThresholdSelector` | `pyspark.ml.feature` | Feature — Estimator | Low-variance pruning |
| `DCT` | `pyspark.ml.feature` | Feature — Transformer | Discrete cosine transform |
| `RFormula` | `pyspark.ml.feature` | Feature — Estimator | R-style formula |
| `SQLTransformer` | `pyspark.ml.feature` | Feature — Transformer | SQL pipeline stage |
| `BucketedRandomProjectionLSH` | `pyspark.ml.feature` | Feature — Estimator | ANN, Euclidean |
| `MinHashLSH` | `pyspark.ml.feature` | Feature — Estimator | ANN, Jaccard |
| `LogisticRegression` | `pyspark.ml.classification` | Model — Estimator | Binary/multinomial classification |
| `DecisionTreeClassifier` | `pyspark.ml.classification` | Model — Estimator | Tree classification |
| `RandomForestClassifier` | `pyspark.ml.classification` | Model — Estimator | Bagged trees |
| `GBTClassifier` | `pyspark.ml.classification` | Model — Estimator | Boosted trees (binary) |
| `NaiveBayes` | `pyspark.ml.classification` | Model — Estimator | Probabilistic classifier |
| `LinearSVC` | `pyspark.ml.classification` | Model — Estimator | Linear SVM (binary) |
| `MultilayerPerceptronClassifier` | `pyspark.ml.classification` | Model — Estimator | Feed-forward NN |
| `OneVsRest` | `pyspark.ml.classification` | Model — Meta-estimator | Binary → multiclass |
| `FMClassifier` | `pyspark.ml.classification` | Model — Estimator | Factorization machines |
| `LinearRegression` | `pyspark.ml.regression` | Model — Estimator | OLS / regularized regression |
| `GeneralizedLinearRegression` | `pyspark.ml.regression` | Model — Estimator | GLM families |
| `DecisionTreeRegressor` | `pyspark.ml.regression` | Model — Estimator | Tree regression |
| `RandomForestRegressor` | `pyspark.ml.regression` | Model — Estimator | Bagged trees |
| `GBTRegressor` | `pyspark.ml.regression` | Model — Estimator | Boosted trees |
| `AFTSurvivalRegression` | `pyspark.ml.regression` | Model — Estimator | Survival analysis |
| `IsotonicRegression` | `pyspark.ml.regression` | Model — Estimator | Monotonic regression |
| `FMRegressor` | `pyspark.ml.regression` | Model — Estimator | Factorization machines |
| `KMeans` | `pyspark.ml.clustering` | Model — Estimator | Hard clustering |
| `BisectingKMeans` | `pyspark.ml.clustering` | Model — Estimator | Divisive hierarchical clustering |
| `GaussianMixture` | `pyspark.ml.clustering` | Model — Estimator | Soft clustering (EM) |
| `LDA` | `pyspark.ml.clustering` | Model — Estimator | Topic modeling |
| `PowerIterationClustering` | `pyspark.ml.clustering` | Model — Transformer-only | Graph clustering |
| `ALS` | `pyspark.ml.recommendation` | Model — Estimator | Collaborative filtering |
| `FPGrowth` | `pyspark.ml.fpm` | Pattern mining — Estimator | Frequent itemsets / rules |
| `PrefixSpan` | `pyspark.ml.fpm` | Pattern mining — Transformer-only | Sequential patterns |
| `BinaryClassificationEvaluator` | `pyspark.ml.evaluation` | Evaluation | AUC-ROC / AUC-PR |
| `MulticlassClassificationEvaluator` | `pyspark.ml.evaluation` | Evaluation | Accuracy / F1 / precision / recall |
| `MultilabelClassificationEvaluator` | `pyspark.ml.evaluation` | Evaluation | Multilabel metrics |
| `RegressionEvaluator` | `pyspark.ml.evaluation` | Evaluation | RMSE / MAE / R² |
| `ClusteringEvaluator` | `pyspark.ml.evaluation` | Evaluation | Silhouette score |
| `RankingEvaluator` | `pyspark.ml.evaluation` | Evaluation | MAP / NDCG / precision@k |
| `ParamGridBuilder` | `pyspark.ml.tuning` | Tuning | Grid construction |
| `CrossValidator` / `CrossValidatorModel` | `pyspark.ml.tuning` | Tuning | k-fold search |
| `TrainValidationSplit` / `TrainValidationSplitModel` | `pyspark.ml.tuning` | Tuning | Single-split search |
| `Correlation` | `pyspark.ml.stat` | Statistics | Correlation matrix |
| `ChiSquareTest` | `pyspark.ml.stat` | Statistics | Independence test |
| `ANOVATest` / `FValueTest` | `pyspark.ml.stat` | Statistics | Continuous-feature significance tests |
| `Summarizer` | `pyspark.ml.stat` | Statistics | Vector column summary stats |
| `KolmogorovSmirnovTest` | `pyspark.ml.stat` | Statistics | Distribution goodness-of-fit |
| `MLReader` / `MLWriter` / `MLReadable` / `MLWritable` | `pyspark.ml.util` | Persistence | Save/load infrastructure |
| `vector_to_array` / `array_to_vector` | `pyspark.ml.functions` | Interop | Vector ⇄ array conversion (Catalyst-native) |
| `predict_batch_udf` | `pyspark.ml.functions` | Interop | Vectorized batch-inference UDF |
| `TorchDistributor` | `pyspark.ml.torch.distributor` | Deep learning | Distributed PyTorch launcher |

---

## 19. End-to-End Worked Example

Binary classification: split → feature pipeline → model → hyperparameter search → evaluate → log to MLflow.

```python
from pyspark.ml import Pipeline
from pyspark.ml.feature import StringIndexer, OneHotEncoder, VectorAssembler, StandardScaler
from pyspark.ml.classification import RandomForestClassifier
from pyspark.ml.evaluation import BinaryClassificationEvaluator
from pyspark.ml.tuning import CrossValidator, ParamGridBuilder
import mlflow, mlflow.spark

# 1. Split (do this before fitting ANY estimator, to avoid leakage)
train_df, test_df = raw_df.randomSplit([0.8, 0.2], seed=42)
train_df.cache()

# 2. Feature pipeline stages
cat_idx  = StringIndexer(inputCols=["plan_type", "region"],
                          outputCols=["plan_idx", "region_idx"],
                          handleInvalid="keep")
cat_ohe  = OneHotEncoder(inputCols=["plan_idx", "region_idx"],
                          outputCols=["plan_ohe", "region_ohe"])
assembler = VectorAssembler(inputCols=["tenure_months", "monthly_spend", "plan_ohe", "region_ohe"],
                             outputCol="features_raw", handleInvalid="keep")
scaler   = StandardScaler(inputCol="features_raw", outputCol="features")

# 3. Model
rf = RandomForestClassifier(featuresCol="features", labelCol="churned", seed=42)

pipeline = Pipeline(stages=[cat_idx, cat_ohe, assembler, scaler, rf])

# 4. Hyperparameter search over the WHOLE pipeline (re-fits scaler/indexer per fold too)
grid = (ParamGridBuilder()
        .addGrid(rf.numTrees, [100, 200])
        .addGrid(rf.maxDepth, [5, 10])
        .build())

evaluator = BinaryClassificationEvaluator(labelCol="churned", metricName="areaUnderROC")

cv = CrossValidator(estimator=pipeline, estimatorParamMaps=grid,
                     evaluator=evaluator, numFolds=5, parallelism=4, seed=42)

mlflow.pyspark.ml.autolog()
with mlflow.start_run(run_name="churn_rf_cv"):
    cv_model = cv.fit(train_df)
    best_pipeline_model = cv_model.bestModel

    test_preds = best_pipeline_model.transform(test_df)
    test_auc = evaluator.evaluate(test_preds)
    mlflow.log_metric("test_auc", test_auc)

    mlflow.spark.log_model(best_pipeline_model, artifact_path="model",
                            registered_model_name="analytics.churn.rf_model")

# 5. Batch scoring later, from the registry
loaded = mlflow.spark.load_model("models:/analytics.churn.rf_model/1")
scored = loaded.transform(new_customers_df)
```

---

*Reference card scope: `pyspark.ml` (DataFrame-based API) only. `pyspark.mllib` (RDD-based, legacy) intentionally excluded — use `pyspark.ml` for all new work.*
