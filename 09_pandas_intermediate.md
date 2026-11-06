# 09 - Pandas Intermediate

Selection, filtering, groupby, joins, and reshaping. This is where you do the bulk
of real data-wrangling work.

```python
import pandas as pd
import numpy as np

df = pd.DataFrame({
    "name": ["Ada", "Alan", "Grace", "Linus", "Guido"],
    "dept": ["Eng", "Eng", "Eng", "Ops", "Ops"],
    "age": [36, 41, 45, 33, 52],
    "salary": [90, 85, 120, 70, 110],
})
```

## 1. Selection: .loc vs .iloc (master this)

```
   .loc  -> label-based   (uses index labels and column names)
   .iloc -> integer-based (uses 0-based positions, like NumPy)

   df.loc[row_labels, col_labels]
   df.iloc[row_positions, col_positions]
```

```python
# .loc: labels (endpoint INCLUSIVE for slices)
df.loc[0]                       # row with index label 0 (a Series)
df.loc[0, "name"]               # single value
df.loc[0:2]                     # rows labeled 0,1,2 (INCLUSIVE of 2!)
df.loc[:, "name":"age"]         # column slice by label (inclusive)
df.loc[[0, 2], ["name", "age"]] # explicit lists
df.loc[df["age"] > 40, "name"]  # boolean rows + column selection

# .iloc: positions (endpoint EXCLUSIVE, like Python/NumPy)
df.iloc[0]                      # first row
df.iloc[0, 1]                   # row 0, col 1
df.iloc[0:2]                    # first two rows (EXCLUSIVE of 2)
df.iloc[:, [0, 2]]              # all rows, columns 0 and 2
df.iloc[-1]                     # last row
```

```
   Slicing endpoint difference (SUBTLE, common bug):
   df.loc[0:2]   -> includes label 2   (3 rows)
   df.iloc[0:2]  -> excludes index 2   (2 rows)
```

Single-value fast access:

```python
df.at[0, "name"]      # scalar by label   (faster than .loc for one cell)
df.iat[0, 1]          # scalar by position (faster than .iloc for one cell)
```

## 2. Boolean filtering (the core of querying)

```python
df[df["age"] > 40]                          # rows where age > 40
df[df["dept"] == "Eng"]
df[(df["age"] > 40) & (df["salary"] > 100)] # AND: & with parentheses
df[(df["dept"] == "Ops") | (df["age"] < 35)]  # OR: |
df[~(df["dept"] == "Eng")]                  # NOT: ~
df[df["dept"].isin(["Eng", "Ops"])]         # membership
df[df["age"].between(35, 50)]               # range (inclusive)
df[df["name"].str.startswith("G")]          # string condition
df[df["salary"].isna()]                     # missing values
```

Use `&`, `|`, `~` (not `and`/`or`/`not`) and wrap each condition in parentheses;
operator precedence otherwise causes errors.

### The .query() method (readable filtering)

```python
df.query("age > 40 and salary > 100")
df.query("dept == 'Eng'")
threshold = 40
df.query("age > @threshold")     # @ references a Python variable
df.query("dept in ['Eng', 'Ops']")
```

`.query` is often more readable for complex conditions and avoids repeating the
DataFrame name. Trade-off: slightly slower and no static checking of column names.

## 3. Modifying data safely (avoid SettingWithCopyWarning)

```python
# WRONG: chained indexing may write to a temporary copy
df[df["age"] > 40]["salary"] = 999          # SettingWithCopyWarning; may not stick

# RIGHT: single .loc call for conditional assignment
df.loc[df["age"] > 40, "salary"] = 999
df.loc[df["dept"] == "Eng", "bonus"] = 10
```

```
   Chained indexing:  df[mask][col] = x
     step 1: df[mask]  -> may be a COPY
     step 2: [col]=x   -> writes to the copy, not df  (silently lost)
   Fix: df.loc[mask, col] = x   (one operation, unambiguous)
```

Always use a single `.loc[rows, cols] = value` for conditional writes.

## 4. Transforming columns

```python
# Vectorized arithmetic (preferred, fast)
df["salary_k"] = df["salary"] * 1000

# map: element-wise on a Series, via function or dict
df["dept_full"] = df["dept"].map({"Eng": "Engineering", "Ops": "Operations"})
df["age_group"] = df["age"].map(lambda a: "senior" if a > 40 else "junior")

# apply: apply a function along an axis of a DataFrame
df["name_len"] = df["name"].apply(len)                # on a Series
df.apply(lambda row: row["age"] + row["salary"], axis=1)   # row-wise (slow)
df[["age", "salary"]].apply(np.mean, axis=0)          # column-wise

# np.where / np.select for vectorized conditionals (fastest)
df["tier"] = np.where(df["salary"] > 100, "high", "low")
df["band"] = np.select(
    [df["salary"] < 80, df["salary"] < 110],
    ["low", "mid"],
    default="high",
)

# replace values
df["dept"].replace({"Eng": "Engineering"})
```

```
   Speed ranking for transforms (fast -> slow):
   vectorized ops / np.where   >   .map / .str / .dt   >   .apply(axis=0)   >   .apply(axis=1)
```

Avoid `.apply(..., axis=1)` (row-wise Python loop) on large data; it is often
10-100x slower than a vectorized equivalent. Reach for it only when no vectorized
form exists.

## 5. GroupBy: split-apply-combine

The most important analytical pattern. Split rows into groups, apply a function
per group, combine results.

```
   split ──> apply ──> combine
   ┌────────────────────────────────────────────┐
   │ dept  salary                                │
   │ Eng    90 ┐                                 │
   │ Eng    85 ├─ group "Eng" ─> mean=98.3 ┐     │
   │ Eng   120 ┘                            ├──> result table
   │ Ops    70 ┐                            │    dept  mean
   │ Ops   110 ┴─ group "Ops" ─> mean=90.0 ┘    Eng   98.3
   │                                             Ops   90.0
   └────────────────────────────────────────────┘
```

```python
df.groupby("dept")["salary"].mean()          # mean salary per dept
df.groupby("dept").size()                    # rows per group
df.groupby("dept")["salary"].agg(["mean", "min", "max", "count"])
df.groupby(["dept", "age_group"])["salary"].sum()   # multi-key -> MultiIndex

# Different aggregation per column
df.groupby("dept").agg(
    avg_salary=("salary", "mean"),           # named aggregation (clean output)
    max_age=("age", "max"),
    headcount=("name", "count"),
)

# transform: return same-shape result aligned to original rows (for feature eng)
df["dept_avg"] = df.groupby("dept")["salary"].transform("mean")
df["salary_vs_dept"] = df["salary"] - df["dept_avg"]

# filter: keep groups meeting a condition
df.groupby("dept").filter(lambda g: g["salary"].mean() > 95)

# apply: arbitrary per-group function
df.groupby("dept").apply(lambda g: g.nlargest(1, "salary"))
```

```
   agg       -> one row per group (reduce)
   transform -> same shape as input (broadcast group result back to rows)
   filter    -> subset of original rows (keep/drop whole groups)
   apply     -> anything (flexible, slowest)
```

`transform` is essential for feature engineering: it lets you attach group-level
statistics (group mean, group rank) back onto each original row.

## 6. Joining and merging

Combine DataFrames by key columns (like SQL joins) or by index.

```python
employees = pd.DataFrame({
    "emp_id": [1, 2, 3],
    "name": ["Ada", "Alan", "Grace"],
    "dept_id": [10, 10, 20],
})
depts = pd.DataFrame({
    "dept_id": [10, 20, 30],
    "dept_name": ["Eng", "Ops", "Sales"],
})

pd.merge(employees, depts, on="dept_id", how="inner")
```

### Join types

```
   how="inner"  keep only keys present in BOTH        (intersection)
   how="left"   keep all LEFT rows, match from right  (most common)
   how="right"  keep all RIGHT rows
   how="outer"  keep all keys from both               (union)
   how="cross"  cartesian product (every pair)

        LEFT              RIGHT
      dept_id           dept_id
        10 ┐              10 ┐
        10 ├── inner ─────10 ┘  -> matched rows only
        20 ┘              20
                          30      -> dropped by inner, kept by outer (NaN)
```

```python
pd.merge(employees, depts, on="dept_id", how="left")      # keep all employees
pd.merge(a, b, left_on="id", right_on="user_id")          # different key names
pd.merge(a, b, on="id", suffixes=("_a", "_b"))            # rename overlapping cols
pd.merge(a, b, left_index=True, right_index=True)         # join on index
a.join(b, how="left")                                     # index-based shorthand

# Validate to catch data problems early
pd.merge(a, b, on="id", validate="one_to_one")   # raises if keys duplicated
# also: "one_to_many", "many_to_one", "many_to_many"

# indicator: show where each row came from
pd.merge(a, b, on="id", how="outer", indicator=True)   # adds _merge column
```

Common bug: a many-to-many merge silently multiplies rows. Use `validate=` to
assert the expected cardinality, and check `df.shape` before/after.

### Concatenation (stacking)

```python
pd.concat([df1, df2])                 # stack rows (axis=0), align columns
pd.concat([df1, df2], axis=1)         # stack columns (align on index)
pd.concat([df1, df2], ignore_index=True)   # renumber the index 0..n
pd.concat([df1, df2], keys=["a", "b"])     # add an outer index level
```

`merge` joins on key values; `concat` glues frames together. Use `concat` to
append rows from multiple files, `merge` to enrich rows with columns from another
table.

## 7. Reshaping: pivot, melt, stack

### Wide vs long (tidy) format

```
   LONG (tidy): one observation per row      WIDE: one entity per row
   date  city  temp                          date  London  Paris
   d1    Lon   10                            d1    10      12
   d1    Par   12          <-- pivot -->      d2    11      13
   d2    Lon   11          <-- melt  -->
   d2    Par   13
```

```python
# pivot: long -> wide
wide = long.pivot(index="date", columns="city", values="temp")

# pivot_table: like pivot but aggregates duplicates
pd.pivot_table(df, index="dept", columns="age_group",
               values="salary", aggfunc="mean", fill_value=0, margins=True)

# melt: wide -> long
long = wide.reset_index().melt(id_vars="date",
                               var_name="city", value_name="temp")

# stack / unstack: move between columns and index levels
df.set_index(["dept", "name"]).unstack("dept")   # index level -> columns
stacked = wide.stack()                            # columns -> index level

# crosstab: frequency cross-tabulation
pd.crosstab(df["dept"], df["age_group"])
pd.crosstab(df["dept"], df["age_group"], normalize="index")  # row proportions
```

```
   pivot / pivot_table:  long ─────> wide   (spread a column into new columns)
   melt / stack:         wide ─────> long   (gather columns into rows)

   Rule of thumb: store/compute in LONG (tidy) form, pivot to WIDE only for
   display or specific model inputs.
```

`pivot` requires unique index/column pairs; `pivot_table` aggregates duplicates
(specify `aggfunc`). Prefer long/tidy format for analysis; it groups, filters, and
plots more cleanly.

## 8. Combining it all: a typical wrangling chain

Method chaining reads top-to-bottom as a pipeline. Wrap in parentheses.

```python
result = (
    df
    .query("age > 30")                              # filter rows
    .assign(salary_k=lambda d: d["salary"] * 1000)  # new column
    .groupby("dept", as_index=False)                # group
    .agg(avg_salary=("salary", "mean"),
         headcount=("name", "count"))
    .sort_values("avg_salary", ascending=False)     # order
    .reset_index(drop=True)
)
```

Chaining avoids intermediate variables and makes the transformation sequence
explicit. Use `lambda d: ...` inside `assign`/`query` to reference the
intermediate frame. Trade-off: harder to inspect intermediate steps while
debugging; break the chain temporarily when diagnosing.

## Exercises

1. From the sample `df`, select the name and salary of everyone in Eng earning
   over 90 using `.loc`, then again using `.query`.
2. Add a column with each person's salary minus their department average
   (`groupby.transform`).
3. Merge an orders table with a customers table (left join) and count orders per
   customer; use `validate=` appropriately.
4. Given monthly sales in wide form (columns = months), melt to long, then pivot
   back.
5. Write a single method chain: filter, add a derived column, group, aggregate
   with named outputs, and sort.

Proceed to `10_pandas_advanced.md`.
