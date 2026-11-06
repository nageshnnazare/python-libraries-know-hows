# 07 - NumPy Advanced

Broadcasting, vectorization strategy, linear algebra, the modern random API,
structured arrays, memory layout, and performance. This is where NumPy becomes a
power tool.

```python
import numpy as np
```

## 1. Broadcasting

Broadcasting is the set of rules NumPy uses to operate on arrays of different
shapes without copying data. It is the single most important advanced concept.

### The rules

Compare shapes element-wise from the **trailing** (rightmost) dimension. Two
dimensions are compatible when they are equal, or one of them is 1. A missing
dimension is treated as 1.

```
   Align shapes to the right, then for each axis:
     equal        -> ok
     one is 1     -> stretch the size-1 side
     otherwise    -> ERROR

   Example: (3, 4) + (4,)
       (3, 4)
       (   4)   <- treated as (1, 4)
     ─────────
       (3, 4)   <- the (1,4) row is broadcast across all 3 rows
```

```python
m = np.ones((3, 4))
row = np.array([10, 20, 30, 40])     # shape (4,)
m + row                              # row added to every row -> (3,4)

col = np.array([[1], [2], [3]])      # shape (3, 1)
m + col                              # col added to every column -> (3,4)

a = np.arange(3).reshape(3, 1)       # (3,1)
b = np.arange(4).reshape(1, 4)       # (1,4)
a + b                                # (3,4) outer-sum grid
```

```
   (3,1) + (1,4) -> (3,4):
       b: [0 1 2 3]
   a:0   0 1 2 3
     1   1 2 3 4
     2   2 3 4 5
   Each cell = a[i] + b[j]. No data was copied; NumPy iterates virtually.
```

### Practical broadcasting patterns

```python
# Standardize columns (subtract per-column mean, divide by per-column std)
X = np.random.default_rng(0).normal(size=(100, 5))
X_std = (X - X.mean(axis=0)) / X.std(axis=0)     # (100,5)-(5,) broadcasts

# Pairwise distances between rows of A (m,d) and B (n,d) -> (m,n)
A = np.random.random((4, 2)); B = np.random.random((3, 2))
diff = A[:, None, :] - B[None, :, :]   # (4,1,2)-(1,3,2) -> (4,3,2)
dists = np.sqrt((diff ** 2).sum(axis=2))   # (4,3)

# Multiplication table
np.arange(1, 6)[:, None] * np.arange(1, 6)[None, :]
```

Broadcasting avoids explicit loops and intermediate copies. When shapes do not
align, reshape with `[:, None]` / `np.newaxis` to insert size-1 axes.

## 2. Vectorization strategy

Vectorization means expressing computation as array operations instead of Python
loops. The mindset shift is the core NumPy skill.

```python
# Anti-pattern: Python loop
result = np.empty(len(a))
for i in range(len(a)):
    result[i] = a[i] ** 2 + 2 * a[i] + 1

# Vectorized: one expression, runs in C
result = a ** 2 + 2 * a + 1
```

```
   LOOP:        interpreter processes one element per iteration (slow)
   VECTORIZED:  a single C loop over the whole buffer, often SIMD (fast)

   for-loop  ──────────────>  ~1x
   vectorized ─────────────>  ~10-100x   (typical speedup)
```

Techniques to eliminate loops:

```python
# Conditionals -> np.where / np.select
np.where(a > 0, "pos", "neg")
conds = [a < 0, a == 0, a > 0]
choices = [-1, 0, 1]
np.select(conds, choices)

# Accumulation -> cumsum/cumprod/ufunc.accumulate
np.cumsum(a); np.cumprod(a); np.add.accumulate(a)

# Boolean counting -> sum of a mask
(a > 5).sum()               # count elements > 5
(a > 5).mean()              # fraction > 5
np.count_nonzero(a > 5)

# Reductions -> built-in axis aggregations, not manual loops
```

When you genuinely cannot vectorize (e.g., irregular per-row logic),
`np.vectorize` gives a convenient wrapper, but note it is a Python loop internally
(no speedup, only convenience). For real speed on custom element-wise code, use
Numba or Cython.

## 3. Sorting, searching, unique

```python
a = np.array([3, 1, 2, 1])
np.sort(a)                 # [1 1 2 3]   (returns a copy)
a.sort()                   # in-place
np.argsort(a)              # indices that would sort a -> [1 3 2 0]

# Sort a 2-D array along an axis
m = np.array([[3, 1], [2, 4]])
np.sort(m, axis=0)         # sort each column
np.sort(m, axis=1)         # sort each row

# Sort rows by one column (using argsort as an index)
data = np.array([[3, 100], [1, 200], [2, 150]])
data[np.argsort(data[:, 0])]     # sort rows by first column

# Searching
np.searchsorted([1, 3, 5, 7], 4)   # 2  (insertion index to keep sorted)
np.nonzero(a > 1)                  # indices where condition True
np.argwhere(a > 1)                 # same, as (N, ndim) coordinates

# Unique
u = np.unique([3, 1, 2, 1, 3])                 # [1 2 3] (sorted)
u, counts = np.unique(a, return_counts=True)   # values + frequencies
```

## 4. Linear algebra

```python
A = np.array([[1, 2], [3, 4]])
B = np.array([[5, 6], [7, 8]])
v = np.array([1, 1])

A @ B                    # matrix multiplication (preferred operator)
np.matmul(A, B)          # same
A * B                    # ELEMENT-WISE (Hadamard), NOT matrix mult!
A @ v                    # matrix-vector product -> [3 7]
np.dot(v, v)             # dot product -> 2

A.T                      # transpose
np.linalg.inv(A)         # inverse
np.linalg.det(A)         # determinant
np.trace(A)              # sum of diagonal
np.linalg.matrix_rank(A)

# Solve Ax = b  (do NOT compute inv(A) @ b; solve is faster and more stable)
b = np.array([1, 2])
x = np.linalg.solve(A, b)

# Eigenvalues/vectors, SVD, QR
vals, vecs = np.linalg.eig(A)
U, S, Vt = np.linalg.svd(A)
Q, R = np.linalg.qr(A)

# Norms
np.linalg.norm(v)            # L2 norm
np.linalg.norm(v, ord=1)     # L1
np.linalg.norm(A, ord="fro") # Frobenius
```

```
   A * B   element-wise:   C[i,j] = A[i,j] * B[i,j]
   A @ B   matrix mult:    C[i,j] = sum_k A[i,k] * B[k,j]

   Shapes for @:  (m,k) @ (k,n) -> (m,n)   (inner dims must match)
```

Numerical stability note: to solve linear systems always prefer
`np.linalg.solve(A, b)` over `np.linalg.inv(A) @ b`. The former is faster and
avoids amplifying floating-point error.

## 5. The modern random API (Generator)

Use `np.random.default_rng()`. The legacy `np.random.seed()`/`np.random.rand()`
global functions still work but the Generator API is the recommended, better
approach (independent streams, faster algorithms, explicit state).

```python
rng = np.random.default_rng(seed=42)     # reproducible generator

rng.random(5)                    # uniform [0,1)
rng.integers(0, 10, size=5)      # ints in [0,10)
rng.normal(loc=0, scale=1, size=(2, 3))   # Gaussian
rng.uniform(-1, 1, size=5)
rng.choice([1, 2, 3], size=10, p=[0.5, 0.3, 0.2])   # weighted sampling
rng.permutation(10)              # shuffled 0..9
rng.shuffle(arr)                 # in-place shuffle

# Other distributions
rng.poisson(lam=3, size=5)
rng.binomial(n=10, p=0.5, size=5)
rng.exponential(scale=2.0, size=5)
```

```
   Legacy (avoid for new code):     Modern (prefer):
   np.random.seed(42)               rng = np.random.default_rng(42)
   np.random.rand(3)                rng.random(3)
   np.random.randint(0,10)          rng.integers(0,10)
   np.random.randn(3)               rng.normal(size=3)
```

Reproducibility: pass an explicit seed to `default_rng`. For parallel work, spawn
independent child generators with `rng.spawn(n)` to avoid correlated streams.

## 6. Structured and record arrays

Arrays with named, typed fields; a lightweight table before reaching for pandas.

```python
dt = np.dtype([("name", "U10"), ("age", "i4"), ("score", "f8")])
people = np.array([("Ada", 36, 9.5), ("Alan", 41, 8.7)], dtype=dt)

people["name"]        # array(['Ada', 'Alan'])
people["age"].mean()  # 38.5
people[people["age"] > 38]   # filter rows
```

In practice, pandas DataFrames (file 08) are almost always more convenient for
tabular data. Structured arrays matter mainly for interop with binary formats and
C code.

## 7. Memory layout: strides, contiguity, C vs Fortran order

An ndarray is a flat buffer plus metadata (shape, dtype, strides). Strides say how
many bytes to step to move one index along each axis.

```python
a = np.arange(12).reshape(3, 4)
a.strides           # (32, 8) for int64: 32 bytes per row, 8 per column
a.flags['C_CONTIGUOUS']   # True: row-major (default)
```

```
   C-order (row-major):   memory = [row0 row1 row2 ...]
        [ 0 1 2 3 | 4 5 6 7 | 8 9 10 11 ]
   Fortran-order (col-major): memory = [col0 col1 ...]
        [ 0 4 8 | 1 5 9 | 2 6 10 | 3 7 11 ]

   strides tell NumPy how far to jump in bytes for +1 along each axis.
   A transpose just swaps strides -> no data moves (that's why .T is a view).
```

Why it matters: iterating/aggregating along the contiguous axis is
cache-friendly and faster. `np.ascontiguousarray(x)` forces a C-contiguous copy
when a downstream operation needs it. `reshape` may need a copy if the array is
not contiguous.

## 8. Advanced indexing recipes

```python
m = np.arange(20).reshape(4, 5)

# Select specific (row, col) pairs -> the diagonal-ish picks
rows = np.array([0, 1, 2])
cols = np.array([1, 3, 4])
m[rows, cols]                    # [m[0,1], m[1,3], m[2,4]] = [1, 8, 14]

# Combine slice + fancy index
m[1:3, [0, 2, 4]]

# np.take / np.put
np.take(m, [0, 5, 10])           # take by flat index
np.take_along_axis(m, np.argsort(m, axis=1), axis=1)   # reorder per row

# Fancy-index assignment with duplicates: last write wins (not accumulate)
a = np.zeros(5)
a[[0, 0, 0]] = [1, 2, 3]         # a[0] == 3 (NOT 6)
np.add.at(a, [0, 0, 0], [1, 2, 3])   # use ufunc.at to ACCUMULATE -> a[0]+=6
```

`np.add.at` (and other `ufunc.at`) perform unbuffered in-place accumulation with
repeated indices, which plain fancy-index assignment cannot do.

## 9. Useful whole-array utilities

```python
np.tile(a, (2, 3))          # repeat the whole array as tiles
np.repeat(a, 3)             # repeat each element 3 times
np.roll(a, 2)               # shift elements circularly
np.diff(a)                  # consecutive differences (a[1:]-a[:-1])
np.gradient(a)              # numerical gradient
np.cumsum(a); np.cumprod(a)
np.meshgrid(x, y)           # coordinate grids for surfaces/vectorized f(x,y)
np.histogram(data, bins=10) # counts + bin edges
np.digitize(data, bins)     # which bin each value falls into
np.allclose(a, b)           # float-tolerant equality (use instead of ==)
np.array_equal(a, b)        # exact element-wise equality
np.einsum("ij,jk->ik", A, B)   # Einstein summation (flexible tensor ops)
```

`np.einsum` deserves a mention for advanced users: it expresses many
matrix/tensor operations (dot, outer, trace, batched matmul, transposes) in one
compact notation and is often both readable and fast.

```python
np.einsum("ii->", A)        # trace
np.einsum("ij->ji", A)      # transpose
np.einsum("ij,ij->", A, A)  # sum of element-wise products
np.einsum("bij,bjk->bik", X, Y)   # batched matrix multiply
```

## 10. Performance checklist

```
   [OK] Vectorize: replace Python loops with array ops.
   [OK] Preallocate output arrays; avoid growing in a loop.
   [OK] Use views (slicing) instead of copies where safe.
   [OK] Choose the smallest sufficient dtype (float32 vs float64) for big data.
   [OK] Operate along the contiguous axis; watch C vs F order.
   [OK] Use in-place ops (a += b, out= argument) to avoid temporaries.
   [OK] Use np.linalg.solve, not inv; use @ for matmul.
   [WARN] np.vectorize is convenience, not speed.
   [WARN] Fancy/boolean indexing copies; basic slicing views.
```

```python
# In-place and out= to avoid allocating temporaries in hot code
np.multiply(a, 2, out=a)     # a *= 2 without a new array
a += b                       # in-place add
```

For CPU-bound custom numeric loops that resist vectorization, consider Numba
(`@njit`) which JIT-compiles Python to machine code, or move to Cython.

## Exercises

1. Without loops, compute the pairwise Euclidean distance matrix between rows of a
   (m,d) and (n,d) array using broadcasting.
2. Standardize each column of a matrix to zero mean and unit variance.
3. Given a 1-D array, replace each element with the average of itself and its two
   neighbors (moving average) using slicing.
4. Use `np.einsum` to compute the batched matrix product of shapes (B,m,k) and
   (B,k,n).
5. Demonstrate that `.T` does not copy data by inspecting `.strides` and `.base`.

Proceed to `08_pandas_fundamentals.md`.
