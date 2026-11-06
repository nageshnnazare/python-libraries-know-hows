# 14 - scikit-learn (Machine Learning)

scikit-learn is the standard library for classical machine learning. Its power is
a consistent API and robust tooling for the full workflow: preprocessing, model
fitting, evaluation, and hyperparameter tuning.

```python
import numpy as np
import pandas as pd
from sklearn.model_selection import train_test_split, cross_val_score, GridSearchCV
from sklearn.preprocessing import StandardScaler, OneHotEncoder
from sklearn.pipeline import Pipeline
from sklearn.compose import ColumnTransformer
```

## 1. The unified estimator API (learn this once, use everywhere)

Every model/transformer follows the same pattern:

```
   estimator.fit(X, y)          learn parameters from training data
   estimator.predict(X)         predict labels/values (models)
   estimator.transform(X)       produce transformed data (transformers)
   estimator.fit_transform(X)   fit then transform (transformers)
   estimator.score(X, y)        default metric (accuracy/R^2)
```

```
   X = features, shape (n_samples, n_features)   2-D array/DataFrame
   y = target,   shape (n_samples,)              1-D labels/values

   ┌──────────── X ────────────┐    ┌── y ──┐
   │ feat1  feat2  feat3        │    │ label │
   │  5.1    3.5    1.4         │    │   0   │
   │  4.9    3.0    1.4         │    │   0   │
   │  6.3    3.3    6.0         │    │   2   │
   └───────────────────────────┘    └───────┘
```

This consistency means once you know one estimator, you know them all. Swapping
`LogisticRegression()` for `RandomForestClassifier()` requires changing one line.

## 2. The ML workflow

```
   ┌──────────┐   ┌───────────┐   ┌────────────┐   ┌──────────┐   ┌──────────┐
   │  Load &  │──>│  Split    │──>│ Preprocess │──>│  Train   │──>│ Evaluate │
   │  explore │   │ train/test│   │  (fit on   │   │  (fit on │   │ (on test │
   │          │   │           │   │   train!)  │   │  train)  │   │  only)   │
   └──────────┘   └───────────┘   └────────────┘   └──────────┘   └──────────┘
                        │                                              │
                        └──────── never let test data leak ───────────┘
```

The cardinal rule: **fit all preprocessing on training data only**, then apply to
test data. Fitting a scaler on the full dataset leaks information from test into
training and inflates your scores. Pipelines (below) enforce this automatically.

## 3. Train/test split

```python
from sklearn.datasets import load_iris
X, y = load_iris(return_X_y=True, as_frame=True)

X_train, X_test, y_train, y_test = train_test_split(
    X, y,
    test_size=0.2,          # 20% held out for testing
    random_state=42,        # reproducibility
    stratify=y,             # preserve class proportions (for classification)
)
```

`stratify=y` keeps the class balance identical in train and test, which matters for
imbalanced data. Always set `random_state` for reproducible splits.

## 4. Preprocessing

```python
# Scaling numeric features (many models need this: SVM, KNN, linear, NN)
scaler = StandardScaler()               # (x - mean) / std, fit on TRAIN
X_train_s = scaler.fit_transform(X_train)
X_test_s = scaler.transform(X_test)     # transform only (reuse train stats)

# Other scalers
from sklearn.preprocessing import MinMaxScaler, RobustScaler
MinMaxScaler()      # scale to [0, 1]
RobustScaler()      # uses median/IQR, resistant to outliers

# Encoding categoricals
from sklearn.preprocessing import OneHotEncoder, OrdinalEncoder, LabelEncoder
OneHotEncoder(handle_unknown="ignore", sparse_output=False)   # nominal -> dummies
OrdinalEncoder()    # ordered categories -> integers
LabelEncoder()      # for the TARGET y only, not features

# Missing values
from sklearn.impute import SimpleImputer
SimpleImputer(strategy="median")        # or "mean", "most_frequent", "constant"
```

```
   Which scaler / encoder:
   StandardScaler  numeric, roughly normal, distance/gradient models
   MinMaxScaler    numeric, bounded range needed
   RobustScaler    numeric with outliers
   OneHotEncoder   nominal categories (no order): color, city
   OrdinalEncoder  ordered categories: low < mid < high
   Tree models     do NOT require scaling (splits are scale-invariant)
```

## 5. Pipelines and ColumnTransformer (do it right)

A `Pipeline` chains preprocessing + model into one estimator, so `fit`/`predict`
apply steps in order and prevent leakage. `ColumnTransformer` applies different
preprocessing to different columns.

```python
numeric_features = ["age", "income"]
categorical_features = ["city", "gender"]

preprocess = ColumnTransformer([
    ("num", Pipeline([
        ("impute", SimpleImputer(strategy="median")),
        ("scale", StandardScaler()),
    ]), numeric_features),
    ("cat", Pipeline([
        ("impute", SimpleImputer(strategy="most_frequent")),
        ("onehot", OneHotEncoder(handle_unknown="ignore")),
    ]), categorical_features),
])

from sklearn.ensemble import RandomForestClassifier
model = Pipeline([
    ("prep", preprocess),
    ("clf", RandomForestClassifier(n_estimators=200, random_state=42)),
])

model.fit(X_train, y_train)          # all steps fit on train only
preds = model.predict(X_test)        # all steps applied to test
```

```
   Pipeline guarantees preprocessing is fit on each CV fold's TRAIN split only:
   raw X ──> [impute] ──> [scale/encode] ──> [model] ──> prediction
             \___________ fit on train fold ___________/
   This is the single most important habit for trustworthy scores.
```

Without a pipeline it is easy to accidentally scale/impute using the whole
dataset, leaking test information. Always wrap preprocessing + model in a pipeline.

## 6. Classification

```python
from sklearn.linear_model import LogisticRegression
from sklearn.ensemble import RandomForestClassifier, GradientBoostingClassifier
from sklearn.svm import SVC
from sklearn.neighbors import KNeighborsClassifier
from sklearn.tree import DecisionTreeClassifier
from sklearn.naive_bayes import GaussianNB

clf = LogisticRegression(max_iter=1000)
clf.fit(X_train, y_train)
y_pred = clf.predict(X_test)
y_proba = clf.predict_proba(X_test)      # class probabilities
```

### Classification metrics

```python
from sklearn.metrics import (accuracy_score, precision_score, recall_score,
                             f1_score, confusion_matrix, classification_report,
                             roc_auc_score, roc_curve)

accuracy_score(y_test, y_pred)
confusion_matrix(y_test, y_pred)
print(classification_report(y_test, y_pred))   # precision/recall/f1 per class
roc_auc_score(y_test, y_proba[:, 1])           # binary AUC
```

```
   Confusion matrix (binary):        Metrics from it:
                predicted             precision = TP / (TP + FP)  "of predicted +, how many right"
              neg     pos             recall    = TP / (TP + FN)  "of actual +, how many found"
   actual neg  TN      FP             f1        = harmonic mean(precision, recall)
          pos  FN      TP             accuracy  = (TP+TN) / total

   Accuracy misleads on imbalanced data (99% negatives -> predict all neg = 99%).
   Use precision/recall/F1/AUC instead. Choose based on cost of FP vs FN:
   - spam filter:   precision matters (don't flag good mail)
   - cancer screen: recall matters (don't miss a case)
```

## 7. Regression

```python
from sklearn.linear_model import LinearRegression, Ridge, Lasso, ElasticNet
from sklearn.ensemble import RandomForestRegressor, GradientBoostingRegressor
from sklearn.svm import SVR

reg = Ridge(alpha=1.0)          # L2 regularization; Lasso = L1 (feature selection)
reg.fit(X_train, y_train)
y_pred = reg.predict(X_test)
reg.coef_, reg.intercept_       # learned linear coefficients
```

### Regression metrics

```python
from sklearn.metrics import (mean_squared_error, mean_absolute_error,
                             r2_score, mean_absolute_percentage_error)

mean_squared_error(y_test, y_pred)                     # MSE
mean_squared_error(y_test, y_pred) ** 0.5              # RMSE (same units as y)
mean_absolute_error(y_test, y_pred)                    # MAE (robust to outliers)
r2_score(y_test, y_pred)                               # 1.0 = perfect, 0 = mean baseline
```

```
   RMSE  penalizes large errors heavily (squared); same units as target
   MAE   average absolute error; robust to outliers
   R^2   fraction of variance explained; can be negative (worse than mean)
   Regularization:
   Ridge (L2)  shrinks coefficients, keeps all features
   Lasso (L1)  drives some coefficients to 0 -> automatic feature selection
```

## 8. Cross-validation (reliable evaluation)

A single train/test split is noisy. K-fold CV averages performance over k splits.

```python
from sklearn.model_selection import cross_val_score, cross_validate, KFold, StratifiedKFold

scores = cross_val_score(model, X, y, cv=5, scoring="accuracy")
scores.mean(), scores.std()      # report mean +/- std

# Multiple metrics at once
cross_validate(model, X, y, cv=5,
               scoring=["accuracy", "f1_macro", "roc_auc_ovr"])

cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)  # for classification
```

```
   5-fold cross-validation:
   fold 1: [TEST ][train][train][train][train]
   fold 2: [train][TEST ][train][train][train]
   fold 3: [train][train][TEST ][train][train]
   fold 4: [train][train][train][TEST ][train]
   fold 5: [train][train][train][train][TEST ]
   Each row trains on 4/5, tests on 1/5; average the 5 scores.
   Use StratifiedKFold for classification to preserve class balance per fold.
```

## 9. Hyperparameter tuning

```python
from sklearn.model_selection import GridSearchCV, RandomizedSearchCV

param_grid = {
    "clf__n_estimators": [100, 200, 500],       # note the step__param syntax
    "clf__max_depth": [None, 10, 20],
    "clf__min_samples_split": [2, 5, 10],
}
search = GridSearchCV(model, param_grid, cv=5, scoring="f1_macro", n_jobs=-1)
search.fit(X_train, y_train)

search.best_params_          # best hyperparameters found
search.best_score_           # best CV score
search.best_estimator_       # refit model with best params
best = search.best_estimator_
best.score(X_test, y_test)   # final evaluation on untouched test set
```

```
   GridSearchCV        exhaustive over the grid; use for small grids
   RandomizedSearchCV  samples n_iter random combos; better for large spaces
   step__param         double-underscore addresses a step inside a Pipeline
```

Tune with CV on the training set, then evaluate the single best model once on the
held-out test set. Do not tune against the test set (that overfits to it).

## 10. Unsupervised learning

```python
# Clustering
from sklearn.cluster import KMeans, DBSCAN, AgglomerativeClustering
km = KMeans(n_clusters=3, random_state=42, n_init=10)
labels = km.fit_predict(X)
km.inertia_                  # within-cluster sum of squares (for elbow method)

# Dimensionality reduction
from sklearn.decomposition import PCA
pca = PCA(n_components=2)
X_2d = pca.fit_transform(X)
pca.explained_variance_ratio_   # variance captured by each component

from sklearn.manifold import TSNE
X_tsne = TSNE(n_components=2, random_state=42).fit_transform(X)  # for viz only
```

```
   PCA    linear, fast, reversible; use to compress/denoise/visualize
   t-SNE  nonlinear, for VISUALIZATION only (distances not meaningful globally)
   KMeans needs n_clusters up front; assumes spherical clusters
   DBSCAN finds arbitrary shapes, detects noise, no k needed; sensitive to eps
```

## 11. Feature importance and inspection

```python
# Tree models expose impurity-based importances
rf = RandomForestClassifier().fit(X_train, y_train)
importances = pd.Series(rf.feature_importances_, index=X_train.columns)
importances.sort_values(ascending=False)

# Permutation importance (model-agnostic, more reliable)
from sklearn.inspection import permutation_importance
r = permutation_importance(model, X_test, y_test, n_repeats=10, random_state=42)

# Linear model coefficients (after scaling, comparable magnitudes)
pd.Series(clf.coef_[0], index=X_train.columns).sort_values()
```

Prefer permutation importance over impurity-based importance: the latter is biased
toward high-cardinality features. Both describe association, not causation.

## 12. Persisting models

```python
import joblib
joblib.dump(model, "model.pkl")        # save the whole fitted pipeline
loaded = joblib.load("model.pkl")
loaded.predict(X_new)
```

Save the entire pipeline (preprocessing + model) so inference applies identical
transforms. Pickle/joblib files are version-sensitive; record the sklearn version.

## 13. Common mistakes

```
   [WARN] Fitting scaler/imputer on full data before split -> data leakage
   [WARN] Tuning hyperparameters on the test set -> optimistic scores
   [WARN] Using accuracy on imbalanced data -> use precision/recall/F1/AUC
   [WARN] Forgetting stratify on imbalanced classification splits
   [WARN] Comparing models on different splits -> use the same CV / random_state
   [WARN] Interpreting feature importance as causation
   [WARN] Not scaling features for distance/gradient models (KNN, SVM, linear)
```

## 14. A complete minimal example

```python
from sklearn.datasets import load_breast_cancer
from sklearn.model_selection import train_test_split, GridSearchCV
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import classification_report

X, y = load_breast_cancer(return_X_y=True, as_frame=True)
X_tr, X_te, y_tr, y_te = train_test_split(X, y, test_size=0.2,
                                          stratify=y, random_state=42)

pipe = Pipeline([("scale", StandardScaler()),
                 ("clf", LogisticRegression(max_iter=5000))])
grid = GridSearchCV(pipe, {"clf__C": [0.1, 1, 10]}, cv=5, scoring="f1")
grid.fit(X_tr, y_tr)

print("best params:", grid.best_params_)
print(classification_report(y_te, grid.predict(X_te)))
```

## Exercises

1. Build a full pipeline (impute + scale numeric, impute + one-hot categorical)
   feeding a `RandomForestClassifier`; evaluate with 5-fold stratified CV.
2. Compare LogisticRegression, RandomForest, and SVC on the same data using
   `cross_val_score`; report mean +/- std for each.
3. Tune `RandomForestClassifier` with `RandomizedSearchCV`, then evaluate the best
   estimator once on the test set.
4. On an imbalanced dataset, show why accuracy is misleading and report
   precision/recall/F1/AUC instead.
5. Reduce a high-dimensional dataset to 2-D with PCA, plot it colored by class, and
   report the explained variance ratio.

Proceed to `15_plotly.md`.
