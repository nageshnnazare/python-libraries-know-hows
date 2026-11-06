# 16 - Workflow, Best Practices, and Capstone

Tying it together: project structure, a repeatable EDA workflow, performance
decisions, reproducibility, common anti-patterns, and an end-to-end capstone that
uses every library in this tutorial.

## 1. Project structure

```
   my_project/
   ├── data/
   │   ├── raw/            <- original, immutable data (never edit by hand)
   │   ├── interim/        <- partially processed
   │   └── processed/      <- final, model-ready
   ├── notebooks/          <- exploration (numbered: 01_eda.ipynb, ...)
   ├── src/                <- reusable, importable code
   │   ├── __init__.py
   │   ├── data.py         <- loading/cleaning functions
   │   ├── features.py     <- feature engineering
   │   ├── models.py       <- training/evaluation
   │   └── viz.py          <- plotting helpers
   ├── outputs/            <- figures, reports, saved models
   ├── tests/
   ├── requirements.txt    <- pinned dependencies
   └── README.md
```

```
   Principle: raw data is sacred and read-only. All transformations are code,
   so any output can be regenerated from raw + code. Notebooks explore;
   src/ holds the code you trust and reuse.
```

The trade-off: this structure is overkill for a throwaway analysis, essential for
anything you will revisit or share. Start light; graduate to it as the project
grows.

## 2. Reproducibility

```python
# 1. Pin dependencies
#    pip freeze > requirements.txt
#    (or use a lockfile: pip-tools, poetry, conda env export)

# 2. Seed every source of randomness
import numpy as np, random
SEED = 42
random.seed(SEED)
rng = np.random.default_rng(SEED)     # pass rng / random_state everywhere
# sklearn: random_state=SEED   train_test_split, models, CV

# 3. Record versions in outputs
import sys, sklearn, pandas as pd
print(sys.version, pd.__version__, sklearn.__version__)

# 4. Prefer scripts/functions over notebook cell-execution order
#    (out-of-order cell runs are the top cause of "works on my machine")
```

```
   Reproducibility checklist:
   [OK] pinned requirements.txt / lockfile
   [OK] fixed random seeds passed explicitly (not global state)
   [OK] raw data versioned or checksummed
   [OK] deterministic pipeline (no reliance on cell run order)
   [OK] recorded library versions with results
```

## 3. A repeatable EDA workflow

```
   1. LOAD & PEEK       df.head(), df.shape, df.info(), df.dtypes
   2. QUALITY           df.isna().sum(), df.duplicated().sum(), df.describe()
   3. UNIVARIATE        distributions: hist/box per numeric; value_counts per cat
   4. BIVARIATE         relationships: scatter, corr(), groupby aggregates
   5. CLEAN             fix dtypes, handle missing, remove/flag outliers, dedup
   6. FEATURE ENGINEER  derive columns, encode categoricals, scale
   7. MODEL (if ML)     split -> pipeline -> CV -> tune -> test-set eval
   8. COMMUNICATE       clear figures + written findings
```

```python
# A compact "first look" function you can reuse
def overview(df):
    print("shape:", df.shape)
    print("\ndtypes:\n", df.dtypes)
    print("\nmissing:\n", df.isna().sum())
    print("\nduplicates:", df.duplicated().sum())
    print("\nnumeric summary:\n", df.describe().T)
    for c in df.select_dtypes("object").columns:
        print(f"\n{c} top values:\n", df[c].value_counts().head())
```

### Handling missing data (decision)

```
   How much missing?
   < 5% and random      -> drop rows (dropna) or simple impute
   moderate, informative-> impute + add is_missing indicator column
   time series          -> ffill/bfill or interpolate()
   whole column mostly  -> consider dropping the column
   Always ask WHY it is missing before choosing.
```

### Handling outliers (decision)

```python
# Detect
z = (df["x"] - df["x"].mean()) / df["x"].std()
outliers = df[z.abs() > 3]                       # z-score method
q1, q3 = df["x"].quantile([0.25, 0.75])
iqr = q3 - q1
mask = (df["x"] < q1 - 1.5*iqr) | (df["x"] > q3 + 1.5*iqr)   # IQR method
```

```
   Outlier response depends on cause:
   data error        -> fix or remove
   genuine extreme   -> keep; use robust methods (median, RobustScaler) or
                        transform (log) instead of deleting real signal
   Never delete outliers just to make a model look better.
```

## 4. Performance decision tree

```
   Is it slow?
     │
     ├─ Is it a Python loop over rows/elements?
     │     └─ YES -> vectorize with NumPy/pandas (biggest win, 10-100x)
     │
     ├─ pandas .apply(axis=1)?
     │     └─ YES -> replace with vectorized ops / groupby.transform / np.where
     │
     ├─ Data bigger than RAM or pandas too slow?
     │     └─ YES -> Polars, DuckDB (SQL on files), or Dask; use Parquet
     │
     ├─ Custom numeric code that can't vectorize?
     │     └─ YES -> Numba (@njit) or Cython
     │
     └─ CPU-bound and parallelizable?
           └─ YES -> ProcessPoolExecutor, or a model's n_jobs=-1

   Rule: measure first (%timeit / cProfile), optimize the actual bottleneck.
```

```python
# Memory reductions that usually pay off on big tabular data:
df["cat_col"] = df["cat_col"].astype("category")     # repeated strings
df["small_int"] = pd.to_numeric(df["small_int"], downcast="integer")
df.to_parquet("data.parquet")                        # typed, compressed, fast
```

## 5. Writing clean data-science code

```
   [OK] Functions over copy-pasted cells; parameterize paths and constants.
   [OK] Vectorize; avoid loops over DataFrame rows.
   [OK] Use pipelines (sklearn) to prevent leakage and enable reuse.
   [OK] Name things clearly; df is fine for one frame, not for five.
   [OK] Validate assumptions: assert shapes, check dtypes, verify no NaN
        where none is expected, use merge validate=.
   [OK] Keep raw data immutable; make every step reproducible from code.
   [OK] Comment WHY (a chosen threshold, a domain quirk), not WHAT.
   [WARN] Silent errors: don't wrap everything in try/except pass.
   [WARN] Chained indexing for assignment (use .loc).
   [WARN] Data leakage: fit preprocessing on train only.
```

```python
# Defensive checks are cheap and catch bugs early
assert df["age"].between(0, 120).all(), "impossible ages present"
assert not df["id"].duplicated().any(), "duplicate ids"
before = len(df); df = df.merge(other, on="id"); assert len(df) == before, "merge grew rows"
```

## 6. The libraries in one picture (recap)

```
   TASK                         REACH FOR
   ────────────────────────────────────────────────────────
   numeric arrays, math         NumPy
   tabular data, wrangling      pandas
   static charts                Matplotlib
   interactive charts / 3-D     Plotly
   stats, optimization, signal  SciPy
   machine learning             scikit-learn
   bigger-than-RAM tables       Polars / DuckDB / Dask (beyond this tutorial)
```

## 7. Capstone: end-to-end mini-project

A complete workflow using every library. Assume a CSV `sales.csv` with columns:
`date, region, product, units, price, customer_age`.

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import plotly.express as px
from scipy import stats
from sklearn.model_selection import train_test_split, cross_val_score
from sklearn.pipeline import Pipeline
from sklearn.compose import ColumnTransformer
from sklearn.preprocessing import StandardScaler, OneHotEncoder
from sklearn.ensemble import RandomForestRegressor
from sklearn.metrics import mean_absolute_error, r2_score

SEED = 42
rng = np.random.default_rng(SEED)

# ---- 0. (Demo) synthesize data so this runs standalone ----
n = 5000
dates = pd.date_range("2023-01-01", periods=n, freq="h")
df = pd.DataFrame({
    "date": rng.choice(dates, n),
    "region": rng.choice(["North", "South", "East", "West"], n),
    "product": rng.choice(["A", "B", "C"], n, p=[0.5, 0.3, 0.2]),
    "units": rng.poisson(5, n) + 1,
    "price": rng.normal(20, 5, n).round(2).clip(1, None),
    "customer_age": rng.integers(18, 75, n),
})
df["revenue"] = df["units"] * df["price"]

# ---- 1. EDA ----
print(df.shape)
print(df.info())
print(df.describe().T)
print("missing:\n", df.isna().sum())

# ---- 2. Feature engineering (pandas + time series) ----
df["month"] = df["date"].dt.month
df["dayofweek"] = df["date"].dt.dayofweek
df["is_weekend"] = df["dayofweek"].isin([5, 6])
# group-level feature via transform (file 09)
df["region_avg_rev"] = df.groupby("region")["revenue"].transform("mean")

# ---- 3. A hypothesis test (SciPy, file 13) ----
weekend = df.loc[df["is_weekend"], "revenue"]
weekday = df.loc[~df["is_weekend"], "revenue"]
t, p = stats.ttest_ind(weekend, weekday, equal_var=False)
print(f"weekend vs weekday revenue: t={t:.2f}, p={p:.4f}, "
      f"diff={weekend.mean() - weekday.mean():.2f}")

# ---- 4. Static viz (matplotlib, files 11-12) ----
fig, axes = plt.subplots(1, 2, figsize=(12, 4))
df.groupby("region")["revenue"].mean().plot(kind="bar", ax=axes[0],
                                            title="Avg revenue by region")
axes[1].hist(df["revenue"], bins=40, edgecolor="black")
axes[1].set_title("Revenue distribution")
fig.tight_layout()
fig.savefig("outputs_capstone.png", dpi=150, bbox_inches="tight")

# ---- 5. Interactive viz (Plotly, file 15) ----
monthly = (df.groupby([df["date"].dt.to_period("M").astype(str), "region"],
                      as_index=False)["revenue"].sum()
             .rename(columns={"date": "month"}))
figpx = px.line(monthly, x="month", y="revenue", color="region",
                title="Monthly revenue by region")
figpx.write_html("outputs_capstone.html")

# ---- 6. Predictive model (scikit-learn, file 14) ----
target = "revenue"
features_num = ["units", "price", "customer_age", "month", "dayofweek"]
features_cat = ["region", "product"]
X = df[features_num + features_cat]
y = df[target]

X_tr, X_te, y_tr, y_te = train_test_split(X, y, test_size=0.2, random_state=SEED)

pre = ColumnTransformer([
    ("num", StandardScaler(), features_num),
    ("cat", OneHotEncoder(handle_unknown="ignore"), features_cat),
])
model = Pipeline([("prep", pre),
                  ("rf", RandomForestRegressor(n_estimators=200,
                                               random_state=SEED, n_jobs=-1))])

cv = cross_val_score(model, X_tr, y_tr, cv=5,
                     scoring="neg_mean_absolute_error")
print(f"CV MAE: {-cv.mean():.2f} +/- {cv.std():.2f}")

model.fit(X_tr, y_tr)
pred = model.predict(X_te)
print(f"Test MAE: {mean_absolute_error(y_te, pred):.2f}, "
      f"R2: {r2_score(y_te, pred):.3f}")

# ---- 7. Persist ----
import joblib
joblib.dump(model, "outputs_capstone_model.pkl")
```

```
   Capstone data flow:
   raw CSV ─> pandas clean/feature ─> SciPy test ─> matplotlib+Plotly viz
                                   └─> sklearn pipeline ─> CV ─> test eval ─> saved model
   NumPy underlies every numeric step.
```

Note: `revenue = units * price` here, so the model's job is trivially easy by
design (it will score near-perfect). In a real project, the target must not be
directly derivable from features, or you have leakage. This is included as a
deliberate teaching point: always ask whether a feature encodes the answer.

## 8. Where to go next

```
   - Deeper stats/modeling:   statsmodels (formulas, inference, time series)
   - Gradient boosting:       XGBoost, LightGBM, CatBoost (tabular SOTA)
   - Deep learning:           PyTorch, JAX, TensorFlow/Keras
   - Big data:                Polars, DuckDB, Dask, Spark
   - Experiment tracking:     MLflow, Weights & Biases
   - Deployment:              FastAPI, Docker; model serving
   - Dashboards:              Dash, Streamlit
```

## Final principles

```
   1. Correctness before speed. Get the right answer, then make it fast.
   2. Vectorize. The interpreter loop is the enemy of numeric performance.
   3. No leakage. Fit preprocessing on train only; keep the test set untouched.
   4. Reproduce everything. Seeds, pinned deps, code-driven pipelines.
   5. Question the data. Missingness, outliers, and "too good" results have causes.
   6. Communicate clearly. A correct analysis nobody understands has no value.
```

## Capstone exercises

1. Replace the synthetic data with a real CSV; adapt the feature list and rerun.
2. Remove the leaking `price`/`units` and predict `revenue` from only region,
   product, time, and age; compare the honest score to the leaky one.
3. Add hyperparameter tuning (`GridSearchCV`) around the pipeline and report the
   best params and test-set performance.
4. Add a permutation-importance plot to explain the model.
5. Turn the monthly-revenue Plotly chart into a small Dash app with a region
   dropdown.

End of tutorial. Revisit any file as a reference; the fastest way to retain this
is to apply it to a dataset you actually care about.
