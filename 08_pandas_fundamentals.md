# 08 - Pandas Fundamentals

Pandas is the standard tool for tabular data in Python. Its two core objects are
the `Series` (1-D labeled array) and the `DataFrame` (2-D labeled table). Both are
built on NumPy but add labeled indexes, heterogeneous columns, and rich missing-
data handling.

```python
import pandas as pd
import numpy as np
```

## 1. The two core objects

```
   Series: a single labeled column          DataFrame: a labeled table
   ┌───────┬───────┐                        ┌───────┬───────┬───────┐
   │ index │ value │                        │ index │  name │  age  │
   ├───────┼───────┤                        ├───────┼───────┼───────┤
   │   a   │  10   │                        │   0   │  Ada  │  36   │
   │   b   │  20   │                        │   1   │  Alan │  41   │
   │   c   │  30   │                        │   2   │  Grace│  45   │
   └───────┴───────┘                        └───────┴───────┴───────┘
                                            A DataFrame is a dict of Series
                                            that share one index.
```

```python
s = pd.Series([10, 20, 30], index=["a", "b", "c"], name="values")
df = pd.DataFrame({
    "name": ["Ada", "Alan", "Grace"],
    "age": [36, 41, 45],
})
```

## 2. Creating Series

```python
pd.Series([1, 2, 3])                       # default RangeIndex 0,1,2
pd.Series([1, 2, 3], index=["x", "y", "z"])
pd.Series({"x": 1, "y": 2})                # from dict (keys -> index)
pd.Series(5, index=["a", "b", "c"])        # scalar broadcast
pd.Series(np.arange(5))                     # from ndarray

s = pd.Series([10, 20, 30], index=["a", "b", "c"])
s.values      # underlying NumPy array
s.index       # Index(['a','b','c'])
s.dtype       # int64
s["b"]        # 20  (label access)
s.iloc[1]     # 20  (positional access)
s * 2         # vectorized, index preserved
```

A Series is a NumPy array plus an index. All NumPy vectorization applies, but
operations align on the index (see alignment below).

## 3. Creating DataFrames

```python
# From a dict of columns (most common)
df = pd.DataFrame({
    "name": ["Ada", "Alan", "Grace"],
    "age": [36, 41, 45],
    "city": ["London", "Manchester", "NYC"],
})

# From a list of dicts (records)
pd.DataFrame([{"a": 1, "b": 2}, {"a": 3, "b": 4}])

# From a 2-D array with explicit labels
pd.DataFrame(np.arange(6).reshape(2, 3),
             columns=["x", "y", "z"], index=["r1", "r2"])

# With a custom index
pd.DataFrame({"age": [36, 41]}, index=["Ada", "Alan"])
```

## 4. Reading and writing data (I/O)

Pandas reads most tabular formats. CSV is the most common.

```python
# CSV
df = pd.read_csv("data.csv")
df = pd.read_csv("data.csv",
                 sep=",",                    # delimiter
                 header=0,                   # row to use as column names
                 index_col="id",             # column to use as the row index
                 usecols=["id", "age"],      # read only some columns
                 dtype={"age": "int32"},     # force dtypes
                 parse_dates=["date"],       # parse date columns
                 na_values=["NA", "?"],      # extra strings to treat as NaN
                 nrows=1000)                 # read only first N rows

df.to_csv("out.csv", index=False)           # write (drop the index column)

# Other formats
pd.read_excel("data.xlsx", sheet_name="Sheet1")   # needs openpyxl
pd.read_json("data.json")
pd.read_parquet("data.parquet")             # columnar, fast, typed (needs pyarrow)
pd.read_sql("SELECT * FROM t", connection)
df.to_parquet("out.parquet")                # prefer parquet for large/typed data
df.to_excel("out.xlsx", index=False)
```

```
   Format choice:
   CSV      universal, human-readable, but untyped and slow for big data
   Parquet  columnar, compressed, preserves dtypes -> best for large pipelines
   Excel    for sharing with non-technical users
   SQL      when data lives in a database
```

For very large CSVs, read in chunks to bound memory:

```python
for chunk in pd.read_csv("huge.csv", chunksize=100_000):
    process(chunk)                          # each chunk is a DataFrame
```

## 5. Inspecting data (always do this first)

```python
df.head()          # first 5 rows (df.head(10) for 10)
df.tail(3)         # last 3 rows
df.sample(5)       # 5 random rows
df.shape           # (rows, columns)
df.columns         # column labels
df.index           # row labels
df.dtypes          # type of each column
df.info()          # dtypes + non-null counts + memory usage
df.describe()      # summary stats for numeric columns
df.describe(include="all")   # include categorical/object columns
df.memory_usage(deep=True)   # per-column memory (deep=True counts object strings)
```

`df.info()` and `df.describe()` are your first two commands on any new dataset:
they reveal dtypes, missing values, and distributions at a glance.

```python
# Quick health checks
df.isna().sum()            # missing count per column
df.nunique()               # unique values per column
df["city"].value_counts()  # frequency table for a column
df["city"].value_counts(normalize=True)   # as proportions
df.duplicated().sum()      # number of duplicate rows
```

## 6. The Index

The index is what makes pandas more than NumPy: it labels rows and enables
alignment, fast lookup, and joins.

```python
df.set_index("name")            # promote a column to the index
df.reset_index()                # move the index back to a column (default 0..n)
df.set_index("name", inplace=False)   # returns a new DataFrame by default
df.reindex(["Ada", "Zoe"])      # conform to new index (missing -> NaN)
df.rename(columns={"age": "years"})    # rename columns
df.rename(index={0: "first"})          # rename index labels
df.index.name = "person"        # name the index
```

The index need not be unique or numeric. A meaningful index (dates, IDs) makes
selection and alignment far more powerful.

## 7. Automatic alignment (a defining feature)

Arithmetic between pandas objects aligns on the index/columns first, matching by
label, not position. Non-overlapping labels yield NaN.

```python
a = pd.Series([1, 2, 3], index=["x", "y", "z"])
b = pd.Series([10, 20, 30], index=["y", "z", "w"])
a + b
# w     NaN     <- only in b
# x     NaN     <- only in a
# y    12.0     <- 2 + 10
# z    23.0     <- 3 + 20
```

```
   a:  x=1  y=2  z=3
   b:        y=10 z=20  w=30
              │    │
   a+b: aligned by LABEL -> y=12, z=23; x and w unmatched -> NaN
```

Use fill values to avoid NaN:

```python
a.add(b, fill_value=0)     # treats missing labels as 0
```

This label alignment prevents a whole class of "off-by-one row" bugs, but
surprises people who expect positional NumPy behavior.

## 8. Selecting columns and rows (overview)

Full detail is in file 09; the essentials:

```python
df["age"]                  # a single column -> Series
df[["name", "age"]]        # multiple columns -> DataFrame (note double brackets)
df.age                     # attribute access (only for valid identifiers; avoid)

# Row selection: use .loc (labels) and .iloc (positions)
df.loc[0]                  # row with index label 0
df.iloc[0]                 # first row by position
df.loc[0, "age"]           # scalar by label
df.iloc[0, 1]              # scalar by position
df.loc[df["age"] > 40]     # boolean filtering
```

Always prefer `.loc`/`.iloc` for row selection; plain `df[...]` is ambiguous and
best reserved for column selection and boolean masks.

## 9. Adding, modifying, dropping columns

```python
df["senior"] = df["age"] > 40          # new column from a vectorized expression
df["age_months"] = df["age"] * 12
df["const"] = 1                        # scalar broadcast to all rows

# assign() returns a new DataFrame (good for method chaining)
df2 = df.assign(decade=df["age"] // 10 * 10)

# Drop
df.drop(columns=["const"])             # drop column(s), returns new df
df.drop(index=[0, 1])                  # drop rows by label
df.drop(columns=["x"], inplace=True)   # modify in place (returns None)

# Reorder / rename
df = df[["name", "age", "city"]]       # reorder by selecting in order
```

Note: prefer reassignment (`df = df.drop(...)`) or `assign` over `inplace=True`.
`inplace` rarely improves performance and breaks method chaining; the pandas team
discourages it.

## 10. Basic operations and vectorized methods

```python
df["age"].sum(); df["age"].mean(); df["age"].median()
df["age"].min(); df["age"].max(); df["age"].std()
df["age"].quantile(0.9)
df.sum(numeric_only=True)      # column-wise aggregation
df.mean(numeric_only=True)

# String methods via the .str accessor (vectorized)
df["name"].str.upper()
df["name"].str.lower()
df["name"].str.len()
df["name"].str.contains("a")
df["name"].str.startswith("A")
df["name"].str.replace("a", "@", regex=False)
df["city"].str.split(" ")

# Sorting
df.sort_values("age")                       # by column, ascending
df.sort_values("age", ascending=False)
df.sort_values(["city", "age"])             # multi-key
df.sort_index()                             # by the index
df.nlargest(3, "age"); df.nsmallest(3, "age")
```

The `.str`, `.dt`, and `.cat` accessors expose vectorized string, datetime, and
categorical methods respectively. Prefer them over `.apply(lambda ...)` for speed.

## 11. Missing data basics

```python
df.isna()                    # element-wise boolean mask of missing
df.isna().sum()              # count per column
df.notna()                   # inverse
df.dropna()                  # drop rows with ANY NaN
df.dropna(subset=["age"])    # drop only where 'age' is NaN
df.dropna(axis=1)            # drop columns containing NaN
df.fillna(0)                 # replace NaN with a value
df["age"].fillna(df["age"].median())   # impute with the median
df.fillna(method="ffill")    # forward-fill (carry last valid value)  [deprecated alias: use df.ffill()]
df.ffill(); df.bfill()       # forward / backward fill
```

```
   NaN handling decision:
   - Missing at random, few rows?      -> dropna
   - Need to keep all rows?            -> fillna / impute (mean/median/mode)
   - Time series?                      -> ffill/bfill or interpolate()
   - Missingness is informative?       -> add an "is_missing" indicator column
```

`NaN` is a float; a column with missing integers becomes float64. Pandas also has
the newer nullable dtypes (`Int64`, `boolean`, `string`) and `pd.NA` that preserve
integer types with missing values (file 10).

## 12. dtypes in pandas

```python
df.dtypes
df["age"] = df["age"].astype("int32")       # downcast
df["flag"] = df["flag"].astype("bool")
df["cat"] = df["cat"].astype("category")    # memory-efficient for repeated strings
df["id"] = df["id"].astype("string")        # nullable string dtype
pd.to_numeric(df["x"], errors="coerce")     # parse, bad values -> NaN
pd.to_datetime(df["date"])                  # parse dates
```

```
   object   catch-all (usually Python strings); slow, memory-heavy
   category use for low-cardinality repeated strings -> big memory savings
   int64/float64  numeric defaults
   Int64 (capital)  nullable integer (supports pd.NA)
   datetime64[ns]   timestamps
   bool     True/False
```

Converting repeated-string columns to `category` can cut memory by 10x+ and speed
up groupby/merge. Do this early on large datasets.

## Exercises

1. Load a CSV, print `shape`, `dtypes`, `info()`, and the count of missing values
   per column.
2. Create a DataFrame of 5 people with name/age/city; add a boolean `senior`
   column and a `city_upper` column via `.str`.
3. Demonstrate index alignment: add two Series with partially overlapping indexes
   and explain the NaNs.
4. Convert a low-cardinality string column to `category` and compare memory usage
   before/after with `memory_usage(deep=True)`.
5. Read a large CSV in chunks and compute the total row count without loading it
   all into memory.

Proceed to `09_pandas_intermediate.md`.
