# 06 - NumPy Fundamentals

NumPy is the foundation of the entire scientific Python stack. Its core object is
the `ndarray`: a fixed-size, homogeneous, N-dimensional array stored in a
contiguous block of memory. Everything (pandas, scikit-learn, matplotlib) is built
on it.

```python
import numpy as np
```

## 1. Why NumPy: the performance story

```python
# Pure Python: element-by-element in the interpreter
py_list = list(range(1_000_000))
result = [x * 2 for x in py_list]          # slow: interpreter loop, boxed ints

# NumPy: vectorized, runs in C over contiguous memory
arr = np.arange(1_000_000)
result = arr * 2                            # ~50-100x faster, no Python loop
```

```
   Python list                     NumPy ndarray
   ┌───┬───┬───┬───┐               ┌───┬───┬───┬───┐
   │ * │ * │ * │ * │  pointers     │ 1 │ 2 │ 3 │ 4 │  raw values, contiguous
   └─┬─┴─┬─┴─┬─┴─┬─┘               └───┴───┴───┴───┘
     v   v   v   v                 - one dtype for all elements
   int int int int                 - no per-element object overhead
   (scattered objects,             - CPU cache friendly
    each boxed)                     - operations run in compiled C (SIMD)
```

Two reasons NumPy is fast: (1) data is a contiguous typed buffer (cache-friendly,
no boxing), and (2) operations are vectorized, executed in optimized C loops
instead of the Python interpreter.

## 2. Creating arrays

```python
# From Python sequences
a = np.array([1, 2, 3])                 # 1-D, dtype inferred int64
b = np.array([[1, 2, 3], [4, 5, 6]])    # 2-D (2x3)
f = np.array([1, 2, 3], dtype=np.float64)   # force dtype

# Filled arrays (give a shape tuple)
np.zeros((2, 3))          # 2x3 of 0.0
np.ones((2, 3))           # 2x3 of 1.0
np.full((2, 2), 7)        # 2x2 of 7
np.empty((2, 2))          # uninitialized (fast, contains garbage)
np.eye(3)                 # 3x3 identity matrix
np.zeros_like(b)          # same shape/dtype as b, filled with 0

# Ranges
np.arange(0, 10, 2)       # [0 2 4 6 8]  (like range, but array; can use floats)
np.linspace(0, 1, 5)      # [0. 0.25 0.5 0.75 1.] (5 evenly spaced, endpoint incl)
np.logspace(0, 3, 4)      # [1. 10. 100. 1000.] (log scale)

# Random (see file 07 for the modern Generator API)
rng = np.random.default_rng(42)
rng.random((2, 3))        # uniform [0,1)
rng.integers(0, 10, size=5)
rng.normal(0, 1, size=(2, 2))
```

`arange` vs `linspace`: use `arange` when you know the step; use `linspace` when
you know the *count* of points (and want the endpoint included). For floats,
prefer `linspace` to avoid step-rounding surprises.

## 3. Array attributes (the mental model)

```python
a = np.array([[1, 2, 3], [4, 5, 6]], dtype=np.int32)
a.ndim        # 2         number of dimensions (axes)
a.shape       # (2, 3)    size along each axis
a.size        # 6         total number of elements
a.dtype       # int32     element data type
a.itemsize    # 4         bytes per element
a.nbytes      # 24        total bytes (size * itemsize)
a.T           # transpose (view): shape (3, 2)
```

```
   shape (2, 3):        axis 0 (rows) length 2
                        axis 1 (cols) length 3
        axis 1 ─────────────>
   axis  ┌───┬───┬───┐
    0 │  │ 1 │ 2 │ 3 │   row 0
    │  │  ├───┼───┼───┤
    v  │  │ 4 │ 5 │ 6 │   row 1
       └───┴───┴───┘
```

`axis` is the concept to internalize: axis 0 runs down the rows, axis 1 across the
columns. Aggregations take an `axis` argument that says which axis to collapse.

## 4. Data types (dtypes)

Arrays are homogeneous: all elements share one dtype. Choosing dtypes controls
memory and precision.

```python
np.array([1, 2, 3]).dtype           # int64 (platform default int)
np.array([1.0, 2.0]).dtype          # float64
np.array([1, 2], dtype=np.int8)     # -128..127, 1 byte each
np.array([1, 2], dtype=np.float32)  # half the memory of float64

# Common dtypes
# int8/16/32/64, uint8/16/32/64, float16/32/64, complex64/128, bool_
a.astype(np.float64)                # convert (returns a new array/copy)
```

```
   dtype        bytes  range/notes
   ───────────────────────────────────────────
   int8           1    -128 .. 127
   int32          4    ~ +-2.1e9
   int64          8    ~ +-9.2e18  (default int)
   float32        4    ~7 sig digits
   float64        8    ~15 sig digits (default float)
   bool_          1    True/False
```

Overflow is silent for integers (wraps around); watch for it with small int
types. Downcasting `float64 -> float32` halves memory but loses precision, a common
optimization for large ML feature matrices.

```python
np.array([127], dtype=np.int8) + 1   # -> [-128]  (silent overflow!)
```

## 5. Indexing and slicing

Basic slicing follows the same `[start:stop:step]` rules as Python, extended per
axis.

```python
a = np.arange(10)          # [0 1 2 3 4 5 6 7 8 9]
a[2]                       # 2
a[-1]                      # 9
a[2:5]                     # [2 3 4]
a[::2]                     # [0 2 4 6 8]
a[::-1]                    # reversed

# 2-D: index per axis, separated by comma
m = np.arange(12).reshape(3, 4)
# [[ 0  1  2  3]
#  [ 4  5  6  7]
#  [ 8  9 10 11]]
m[1, 2]        # 6         row 1, col 2   (preferred over m[1][2])
m[0]           # [0 1 2 3] entire row 0
m[:, 1]        # [1 5 9]   entire column 1
m[0:2, 1:3]    # [[1 2],[5 6]]  sub-block
m[:, ::2]      # every other column
```

```
   m[1, 2] selects a single element:
        col: 0  1  2  3
   row 0  [ 0  1  2  3 ]
   row 1  [ 4  5 [6] 7 ]   <- m[1, 2] = 6
   row 2  [ 8  9 10 11 ]

   m[:, 1] selects a whole column -> [1, 5, 9]
```

### Views vs copies (critical)

Basic slicing returns a **view**: a window into the same memory. Modifying a view
modifies the original. This is a frequent source of bugs and a key performance
feature.

```python
a = np.arange(10)
sub = a[2:5]          # VIEW, shares memory
sub[0] = 99
a                     # [0 1 99 3 4 5 6 7 8 9]  (original changed!)

sub = a[2:5].copy()   # explicit COPY, independent
sub[0] = 0
a                     # unchanged

a[2:5].base is a      # True  -> confirms it is a view of a
```

```
   a:   [ 0  1  2  3  4  5  6  7  8  9 ]
                 └──view──┘
   sub = a[2:5]  points into the SAME buffer
   sub[0]=99  ->  writes through to a
```

Fancy indexing (next) and boolean indexing always return copies; basic slicing
returns views. When in doubt, `.copy()`.

### Fancy (integer array) indexing -> copy

```python
a = np.array([10, 20, 30, 40, 50])
a[[0, 2, 4]]              # [10 30 50]  pick specific indices
a[[0, 2, 4]] = 0         # assign to those positions
idx = np.array([[0, 1], [2, 3]])
a[idx]                   # result takes the SHAPE of the index array
```

### Boolean (mask) indexing -> copy

The workhorse for filtering.

```python
a = np.array([1, -2, 3, -4, 5])
mask = a > 0             # [ True False  True False  True]
a[mask]                  # [1 3 5]     keep where True
a[a > 0]                 # same, inline
a[a < 0] = 0             # [1 0 3 0 5] conditional assignment
a[(a > 1) & (a < 5)]     # combine with & (and), | (or), ~ (not)
```

```
   a         = [ 1  -2   3  -4   5 ]
   a > 0     = [ T   F   T   F   T ]   <- boolean mask
   a[a > 0]  = [ 1       3       5 ]   <- keep only True positions
```

Use `&`, `|`, `~` (bitwise) with parentheses around each condition; the Python
keywords `and`/`or`/`not` do NOT work element-wise and will raise.

## 6. Reshaping and reordering

```python
a = np.arange(12)
a.reshape(3, 4)          # 3x4 (product must equal size: 3*4=12)
a.reshape(3, -1)         # -1 = infer this axis -> 3x4
a.reshape(-1)            # flatten to 1-D (view when possible)
a.ravel()                # flatten (view if contiguous)
a.flatten()              # flatten (always a copy)

m = np.arange(6).reshape(2, 3)
m.T                      # transpose -> 3x2 (view)
m.transpose()            # same
np.swapaxes(m, 0, 1)     # swap two axes

# Add/remove size-1 axes (important for broadcasting, file 07)
v = np.array([1, 2, 3])   # shape (3,)
v[:, np.newaxis]          # shape (3, 1) column vector
v[np.newaxis, :]          # shape (1, 3) row vector
v.reshape(3, 1)           # same as [:, np.newaxis]
np.expand_dims(v, 0)      # shape (1, 3)
v2 = v.reshape(1, 3)
v2.squeeze()              # remove size-1 axes -> shape (3,)
```

`reshape` returns a view when possible (no data copy). `-1` lets NumPy compute one
dimension automatically.

## 7. Combining and splitting arrays

```python
a = np.array([[1, 2], [3, 4]])
b = np.array([[5, 6], [7, 8]])

np.concatenate([a, b], axis=0)   # stack rows -> 4x2
np.concatenate([a, b], axis=1)   # stack cols -> 2x4
np.vstack([a, b])                # vertical  (axis 0)
np.hstack([a, b])                # horizontal(axis 1)
np.stack([a, b], axis=0)         # NEW axis -> shape (2, 2, 2)

np.split(np.arange(9), 3)        # 3 equal parts
np.hsplit(m, 2)                  # split columns
np.vsplit(m, 2)                  # split rows
```

```
   vstack (axis 0):        hstack (axis 1):
   ┌─────┐                 ┌─────┬─────┐
   │  a  │                 │  a  │  b  │
   ├─────┤                 └─────┴─────┘
   │  b  │
   └─────┘
```

## 8. Element-wise operations and universal functions (ufuncs)

Arithmetic and math apply element-wise, in C, with no Python loop.

```python
a = np.array([1, 2, 3, 4])
a + 10                   # [11 12 13 14]  (scalar broadcast)
a * 2                    # [2 4 6 8]
a ** 2                   # [1 4 9 16]
a + a                    # [2 4 6 8]      (element-wise)
a / a                    # [1. 1. 1. 1.]

# ufuncs: vectorized math
np.sqrt(a)               # element-wise square root
np.exp(a); np.log(a); np.log10(a)
np.sin(a); np.cos(a)
np.abs([-1, -2])         # [1 2]
np.round(np.array([1.4, 1.6]))   # [1. 2.]
np.clip(a, 2, 3)         # [2 2 3 3]  (limit to [2,3])
np.maximum(a, 2)         # element-wise max vs 2
np.where(a > 2, a, 0)    # [0 0 3 4]  (vectorized if/else)
```

`np.where(condition, x, y)` is the vectorized ternary: pick from `x` where the
condition is True, else from `y`. It is one of the most useful NumPy functions.

## 9. Aggregations and the axis argument

```python
m = np.array([[1, 2, 3],
              [4, 5, 6]])

m.sum()               # 21    (all elements)
m.sum(axis=0)         # [5 7 9]     collapse rows -> per-column sums
m.sum(axis=1)         # [6 15]      collapse cols -> per-row sums
m.mean(); m.std(); m.var()
m.min(); m.max()
m.argmin(); m.argmax()   # index of min/max (flattened)
m.cumsum()            # running sum
m.prod()              # product
np.median(m); np.percentile(m, 75)
```

The mental model for `axis`: it names the axis that gets **collapsed/removed**.

```
   m (shape 2x3):        axis=0 collapses DOWN the rows:
   [[1 2 3]                 sum(col0)=1+4=5, etc.  -> [5 7 9]  (shape 3,)
    [4 5 6]]
                         axis=1 collapses ACROSS the columns:
                            sum(row0)=1+2+3=6, etc. -> [6 15]  (shape 2,)

        axis=0  ↓ ↓ ↓
              [ 1 2 3 ]  ->
       axis=1 [ 4 5 6 ]  ->
```

Use `keepdims=True` to retain the collapsed axis as size 1 (handy for
broadcasting, file 07):

```python
col_means = m.mean(axis=0, keepdims=True)   # shape (1, 3) instead of (3,)
centered = m - col_means                     # broadcasts cleanly
```

## 10. NaN handling

`np.nan` is a float "not a number" that propagates through most math.

```python
a = np.array([1.0, np.nan, 3.0])
a.sum()               # nan  (nan poisons the result)
np.nansum(a)          # 4.0  (ignore nan)
np.nanmean(a)         # 2.0
np.isnan(a)           # [False True False]
a[~np.isnan(a)]       # [1. 3.]  drop nans
np.nan == np.nan      # False!  never compare nan with ==; use np.isnan
```

Integers cannot hold NaN; introducing missing values forces a float dtype. Pandas
(file 08) provides richer missing-data handling.

## 11. Saving and loading

```python
np.save("arr.npy", a)              # binary, single array
loaded = np.load("arr.npy")
np.savez("many.npz", a=a, b=b)     # multiple arrays
np.savetxt("a.csv", m, delimiter=",")   # text (slower, human-readable)
np.loadtxt("a.csv", delimiter=",")
```

Prefer `.npy`/`.npz` for speed and exactness; use text formats only for
interchange.

## Exercises

1. Create a 5x5 matrix with values 1..25 and extract the central 3x3 block.
2. Given a 1-D array, replace all negatives with 0 two ways: boolean masking and
   `np.where`. Confirm one is a copy.
3. Build a 10x10 checkerboard of 0/1 using slicing.
4. Demonstrate the view-vs-copy difference: slice, modify, and show the original
   changed; then repeat with `.copy()`.
5. Compute per-row and per-column means of a 4x3 matrix and explain the resulting
   shapes.

Proceed to `07_numpy_advanced.md`.
