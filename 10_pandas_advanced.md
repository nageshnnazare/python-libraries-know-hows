# 10 - Pandas Advanced

MultiIndex, time series, categoricals, nullable dtypes, window functions, and
performance. These topics separate competent from expert pandas use.

```python
import pandas as pd
import numpy as np
```

## 1. MultiIndex (hierarchical indexing)

A MultiIndex has multiple levels on an axis, letting you represent higher-
dimensional data in a 2-D DataFrame.

```python
arrays = [
    ["Eng", "Eng", "Ops", "Ops"],
    ["Ada", "Alan", "Linus", "Guido"],
]
idx = pd.MultiIndex.from_arrays(arrays, names=["dept", "name"])
df = pd.DataFrame({"salary": [90, 85, 70, 110], "age": [36, 41, 33, 52]},
                  index=idx)
```

```
   MultiIndex on rows:
   dept   name
   Eng    Ada     90  36
          Alan    85  41
   Ops    Linus   70  33
          Guido  110  52
   Level 0 = dept, Level 1 = name.
```

### Selecting with a MultiIndex

```python
df.loc["Eng"]                     # all rows under dept "Eng"
df.loc[("Eng", "Ada")]            # a specific (dept, name) tuple
df.loc["Eng"].loc["Ada"]          # step-by-step
df.xs("Ada", level="name")        # cross-section: select by an inner level
df.loc[(slice(None), "Ada"), :]   # all depts, name == Ada (slice(None) = ":")

# IndexSlice makes multi-level slicing readable
idx = pd.IndexSlice
df.loc[idx[:, "Ada"], :]
df.loc[idx["Eng", :], "salary"]
```

### Creating and moving levels

```python
df.groupby(["dept", "age_group"]).sum()   # groupby with 2 keys -> MultiIndex
df.set_index(["dept", "name"])            # promote two columns
df.reset_index()                          # flatten back to columns
df.reset_index(level="name")              # flatten just one level
df.swaplevel("dept", "name")              # reorder levels
df.sort_index()                           # sort by index (needed for fast slicing)
df.unstack("dept")                        # move a row level to columns
```

MultiIndex frames should be `sort_index()`-ed before slicing; unsorted MultiIndex
slicing is slow and may raise `UnsortedIndexError`. MultiIndex is powerful but
adds complexity; for many tasks a flat frame with regular columns is simpler.

## 2. Time series

Pandas has first-class datetime support: parsing, indexing, resampling, rolling,
and timezone handling. This is one of its biggest strengths.

### Timestamps, ranges, parsing

```python
pd.Timestamp("2024-01-15")
pd.to_datetime("2024-01-15 10:30")
pd.to_datetime(["2024-01-01", "2024-02-01"])
pd.to_datetime(df["date"], format="%Y-%m-%d")     # explicit format = faster/safer
pd.to_datetime(df["date"], errors="coerce")       # bad values -> NaT

# Generate date ranges
pd.date_range("2024-01-01", periods=12, freq="MS")   # month starts
pd.date_range("2024-01-01", "2024-12-31", freq="D")  # daily
# freq codes: D day, W week, MS month-start, ME month-end, h hour, min minute,
#             QS quarter-start, YS year-start, B business day
```

### DatetimeIndex and the .dt accessor

```python
ts = pd.Series(np.arange(365),
               index=pd.date_range("2024-01-01", periods=365, freq="D"))

ts["2024-03"]                # partial-string indexing: all of March
ts["2024-03-01":"2024-03-15"]  # date range slice (inclusive)
ts.loc["2024"]               # entire year

# .dt accessor extracts components from a datetime column
df["date"].dt.year
df["date"].dt.month
df["date"].dt.day_name()
df["date"].dt.dayofweek       # Monday=0
df["date"].dt.quarter
df["date"].dt.is_month_end
(df["end"] - df["start"]).dt.days   # timedelta -> integer days
```

### Resampling (time-based groupby)

```
   Resampling changes the time frequency:
   downsample: fine -> coarse (aggregate)   daily -> monthly mean
   upsample:   coarse -> fine (fill/interp) monthly -> daily ffill
```

```python
ts.resample("ME").mean()          # monthly mean (downsample)
ts.resample("W").sum()            # weekly total
ts.resample("QS").agg(["mean", "max"])
ts.resample("h").ffill()          # upsample to hourly, forward-fill

# resample is groupby-over-time; combine with agg like groupby
df.set_index("date").resample("ME").agg(
    total=("sales", "sum"),
    avg=("sales", "mean"),
)
```

### Rolling and expanding windows

```python
ts.rolling(window=7).mean()       # 7-period moving average
ts.rolling(7, min_periods=1).sum()
ts.rolling("30D").mean()          # time-based window (30 days)
ts.expanding().mean()             # cumulative (all data up to each point)
ts.ewm(span=10).mean()            # exponentially weighted moving average
ts.shift(1)                       # lag by one period
ts.diff()                         # period-over-period change
ts.pct_change()                   # percentage change
```

```
   rolling(3).mean() on [1,2,3,4,5]:
   window slides:  [1,2,3]->2   [2,3,4]->3   [3,4,5]->4
   result:         NaN  NaN  2.0  3.0  4.0   (first 2 are NaN)
```

Rolling/EWM are the backbone of time-series feature engineering (moving averages,
volatility, momentum). Timezones: `ts.tz_localize("UTC").tz_convert("US/Eastern")`.

## 3. Categoricals

A `category` dtype stores repeated values as integer codes plus a lookup table.
Big memory and speed wins for low-cardinality string columns.

```python
s = pd.Series(["low", "high", "low", "mid"], dtype="category")
s.cat.categories                  # Index(['high','low','mid'])
s.cat.codes                       # underlying integer codes

# Ordered categoricals enable comparisons and correct sorting
sizes = pd.Categorical(["M", "S", "L", "M"],
                       categories=["S", "M", "L"], ordered=True)
pd.Series(sizes).sort_values()    # sorts S < M < L, not alphabetically
pd.Series(sizes) > "S"            # ordered comparison works

df["dept"] = df["dept"].astype("category")   # convert existing column
```

```
   object column          category column
   ["Eng","Ops","Eng"]    codes:[0,1,0]  categories:["Eng","Ops"]
   stores 3 full strings  stores 3 ints + 2 strings  -> less memory
```

Convert to `category` when a column has few distinct values repeated many times
(e.g., country, status). Avoid for high-cardinality or unique columns (no benefit,
adds overhead).

## 4. Nullable dtypes and pd.NA

Classic pandas uses NaN (a float) for missing, which forces integer columns to
float. Nullable dtypes (`Int64`, `Float64`, `boolean`, `string`) preserve the type
and use `pd.NA`.

```python
s = pd.Series([1, 2, None], dtype="Int64")   # note capital I; stays integer
s + 1                                          # <NA> propagates, dtype preserved
s = pd.Series([True, None, False], dtype="boolean")
s = pd.Series(["a", None, "b"], dtype="string")   # dedicated string dtype

df.convert_dtypes()          # convert whole frame to best nullable dtypes
pd.isna(pd.NA)               # True
```

```
   Classic:  int col + missing -> float64 with NaN  (type changes!)
   Nullable: Int64 col + missing -> Int64 with <NA> (type preserved)
```

Nullable dtypes are the future direction (and back the pandas 2.x Arrow-backed
types). Trade-off: some third-party libs still expect classic NumPy dtypes, so
convert at boundaries. For maximum performance/interop on large data, consider
`dtype_backend="pyarrow"` in readers.

## 5. Advanced apply, pipe, and vectorization

```python
# pipe: chain custom functions in a readable pipeline
def add_tax(df, rate):
    return df.assign(total=df["price"] * (1 + rate))

result = (df.pipe(add_tax, rate=0.2)
            .pipe(lambda d: d[d["total"] > 100]))

# Vectorized string ops beat apply
df["code"].str.extract(r"(\d+)")          # regex capture group -> DataFrame
df["email"].str.split("@").str[1]         # domain part
df["text"].str.contains(r"\berror\b", regex=True)

# map + fillna for lookups with defaults
mapping = {"US": "North America", "FR": "Europe"}
df["region"] = df["country"].map(mapping).fillna("Other")
```

`pipe` lets you insert your own functions into a method chain, keeping the
top-to-bottom pipeline style even for custom logic.

## 6. Window functions and ranking

```python
df["rank"] = df["salary"].rank(method="dense", ascending=False)
df["pct_rank"] = df["salary"].rank(pct=True)
df["dept_rank"] = df.groupby("dept")["salary"].rank(ascending=False)
df["running_total"] = df["sales"].cumsum()
df["cummax"] = df["sales"].cummax()

# Rank within groups (common analytical need)
df["top_in_dept"] = (df.groupby("dept")["salary"]
                       .rank(method="first", ascending=False) == 1)
```

`rank` methods: `average` (default), `min`, `max`, `first`, `dense`. `dense`
leaves no gaps after ties; `first` breaks ties by order of appearance.

## 7. Performance and memory

```python
# 1. Read only what you need
pd.read_csv("f.csv", usecols=[...], dtype={...})

# 2. Downcast numerics
df["x"] = pd.to_numeric(df["x"], downcast="integer")
df["y"] = pd.to_numeric(df["y"], downcast="float")

# 3. Categoricals for repeated strings (often the biggest win)
df["status"] = df["status"].astype("category")

# 4. Measure memory
df.memory_usage(deep=True).sum() / 1e6      # MB

# 5. Avoid row-wise apply; prefer vectorized ops / np.where / groupby.transform
# 6. Avoid growing a DataFrame in a loop; build a list of frames, concat once
frames = [process(chunk) for chunk in chunks]
result = pd.concat(frames, ignore_index=True)
```

```
   Typical memory optimization on a wide CSV:
   object strings -> category:      -70% to -95% on that column
   float64 -> float32:              -50%
   int64  -> int32/int16/int8:      -50% to -87%
   Combined, often 2-10x smaller total.
```

### When pandas is not enough

```
   Data size vs tool:
   < ~1 GB in RAM        pandas (fine)
   fits but slow         pandas + categoricals + pyarrow backend, or Polars
   > RAM (single node)   Polars (streaming), DuckDB (SQL on files), Dask
   distributed cluster   Spark / Dask
```

Polars (a Rust-based DataFrame library with a similar API) and DuckDB (in-process
SQL over Parquet/CSV) are the common next steps when pandas becomes the
bottleneck. They were out of scope per the tutorial plan but are worth knowing
exist.

## 8. Common pitfalls recap

```
   [WARN] Chained indexing for assignment -> use df.loc[mask, col] = val
   [WARN] .apply(axis=1) on big data      -> vectorize or use groupby.transform
   [WARN] Merging without checking cardinality -> use validate=
   [WARN] Forgetting sort_index() on MultiIndex before slicing
   [WARN] Integer column silently becoming float after introducing NaN
   [WARN] Comparing to NaN with == (always False) -> use .isna()
   [WARN] SettingWithCopyWarning -> you are likely writing to a copy
```

## Exercises

1. Build a MultiIndex frame (dept, name) and select all rows for one person across
   departments using `xs`.
2. Given daily sales, resample to monthly totals and add a 3-month rolling mean.
3. Convert a wide monthly frame to a time-indexed series and compute a 7-day EWM.
4. Optimize a wide DataFrame's memory: downcast numerics, convert repeated strings
   to category, and report the before/after size.
5. Rank employees by salary within each department and flag the top earner per
   department.

Proceed to `11_matplotlib_fundamentals.md`.
