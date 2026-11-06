# 13 - SciPy

SciPy builds on NumPy with domain-specific algorithms: statistics, optimization,
interpolation, integration, signal processing, sparse matrices, and advanced
linear algebra. You reach for it when NumPy's basics are not enough.

```python
import numpy as np
from scipy import stats, optimize, interpolate, integrate, linalg, signal, sparse
```

```
   SciPy submodules (import what you need, not "import scipy"):
   scipy.stats        distributions, hypothesis tests, descriptive stats
   scipy.optimize     minimization, curve fitting, root finding
   scipy.interpolate  fill/estimate between known points
   scipy.integrate    numerical integration, ODE solving
   scipy.linalg       richer linear algebra than numpy.linalg
   scipy.signal       filtering, convolution, spectral analysis
   scipy.sparse       memory-efficient sparse matrices
   scipy.spatial      distances, KD-trees, convex hulls
   scipy.fft          fast Fourier transforms
```

## 1. scipy.stats: distributions

Every distribution shares the same interface, which is the key to remembering it.

```python
from scipy import stats

# A distribution object with parameters (loc=mean, scale=std for normal)
dist = stats.norm(loc=100, scale=15)

dist.pdf(115)        # probability density at x=115
dist.cdf(115)        # P(X <= 115)  -> cumulative probability
dist.ppf(0.95)       # inverse CDF: the value with 95% below it (percentile)
dist.sf(115)         # survival: P(X > 115) = 1 - cdf
dist.rvs(size=1000, random_state=42)   # random samples
dist.mean(); dist.std(); dist.var()
dist.interval(0.95)  # central 95% interval
```

```
   The four functions you must know for any distribution:
   pdf/pmf   height of the curve at x         (density / mass)
   cdf       area to the LEFT of x            P(X <= x)
   ppf       inverse cdf: x for a given prob  (quantile / percentile)
   rvs       draw random samples

        pdf                cdf
         /\                    ___/
        /  \               __/
   ____/    \____      ___/
        x                    x
```

Common distributions (same API): `stats.norm`, `stats.t`, `stats.chi2`,
`stats.binom`, `stats.poisson`, `stats.expon`, `stats.uniform`, `stats.gamma`,
`stats.beta`, `stats.lognorm`.

```python
# Discrete distributions use pmf instead of pdf
stats.binom(n=10, p=0.5).pmf(5)     # P(exactly 5 successes)
stats.poisson(mu=3).pmf(2)
```

## 2. scipy.stats: descriptive statistics and fitting

```python
data = stats.norm(50, 10).rvs(1000, random_state=0)

stats.describe(data)             # nobs, minmax, mean, variance, skew, kurtosis
stats.skew(data)                 # asymmetry
stats.kurtosis(data)             # tailedness
stats.mode(data, keepdims=False)
stats.trim_mean(data, 0.1)       # mean after trimming 10% tails (robust)
stats.zscore(data)               # standardize
stats.iqr(data)                  # interquartile range
stats.sem(data)                  # standard error of the mean

# Fit a distribution's parameters to data (MLE)
mu, sigma = stats.norm.fit(data)         # estimate loc, scale
shape, loc, scale = stats.gamma.fit(data)
```

## 3. scipy.stats: hypothesis testing

```
   Decision guide for common tests:
   Compare 2 group means (independent) ────> ttest_ind
   Compare 2 group means (paired)      ────> ttest_rel
   Compare sample mean to a value      ────> ttest_1samp
   Compare 3+ group means              ────> f_oneway (ANOVA)
   Association between 2 categoricals  ────> chi2_contingency
   Check normality                     ────> shapiro / normaltest
   Correlation between 2 numerics      ────> pearsonr / spearmanr
   Non-parametric 2-group             ────> mannwhitneyu / wilcoxon
```

```python
a = stats.norm(0, 1).rvs(100, random_state=1)
b = stats.norm(0.5, 1).rvs(100, random_state=2)

# Two-sample t-test: are the means different?
t_stat, p_value = stats.ttest_ind(a, b)
# p < 0.05 -> reject the null hypothesis (means likely differ)

stats.ttest_rel(before, after)       # paired (same subjects)
stats.ttest_1samp(a, popmean=0)      # vs a hypothesized mean
stats.f_oneway(g1, g2, g3)           # ANOVA: 3+ groups
stats.mannwhitneyu(a, b)             # non-parametric alternative to ttest_ind

# Correlation
r, p = stats.pearsonr(x, y)          # linear correlation
rho, p = stats.spearmanr(x, y)       # rank (monotonic) correlation

# Normality
stats.shapiro(data)                  # good for n < ~5000
stats.normaltest(data)               # D'Agostino-Pearson

# Categorical association
table = np.array([[10, 20], [30, 40]])
chi2, p, dof, expected = stats.chi2_contingency(table)
```

```
   Interpreting a p-value:
   p is P(observing data this extreme | null hypothesis is true).
   p < alpha (e.g. 0.05) -> reject null.  p >= alpha -> fail to reject.
   p does NOT tell you effect size or practical importance.
   Always report an effect size (mean difference, Cohen's d, r) alongside p.
```

Caveat: statistical significance is not practical significance. With large n,
tiny, meaningless differences become "significant". Always pair tests with effect
sizes and confidence intervals.

## 4. scipy.optimize: minimization and curve fitting

```python
from scipy import optimize

# Minimize a scalar function
def f(x):
    return (x - 3) ** 2 + 1
res = optimize.minimize_scalar(f)
res.x        # 3.0 (minimizer)
res.fun      # 1.0 (minimum value)

# Minimize a multivariate function
def rosenbrock(p):
    x, y = p
    return (1 - x)**2 + 100 * (y - x**2)**2
res = optimize.minimize(rosenbrock, x0=[0, 0], method="BFGS")
res.x        # ~[1, 1]

# Curve fitting: fit parameters of a model to data
def model(x, a, b, c):
    return a * np.exp(-b * x) + c
xdata = np.linspace(0, 4, 50)
ydata = model(xdata, 2.5, 1.3, 0.5) + 0.2 * np.random.default_rng(0).normal(size=50)
params, covariance = optimize.curve_fit(model, xdata, ydata, p0=[1, 1, 1])
# params -> best-fit [a, b, c]; np.sqrt(np.diag(covariance)) -> std errors

# Root finding: solve f(x) = 0
optimize.brentq(lambda x: x**2 - 2, 0, 2)   # -> ~1.414 (bracketing, robust)
optimize.fsolve(lambda x: x**2 - 2, x0=1)   # from a starting guess
```

```
   optimize picks:
   minimize_scalar   1-D minimization
   minimize          N-D minimization (BFGS, Nelder-Mead, L-BFGS-B for bounds)
   curve_fit         fit model params to (x, y) data (least squares)
   brentq/fsolve     find where f(x) = 0
   linprog           linear programming
```

`curve_fit` is the workhorse for fitting nonlinear models; always supply a
reasonable `p0` initial guess for hard problems, and inspect the covariance for
parameter uncertainty.

## 5. scipy.interpolate

Estimate values between known data points.

```python
from scipy import interpolate

x = np.linspace(0, 10, 11)
y = np.sin(x)

f_linear = interpolate.interp1d(x, y, kind="linear")
f_cubic  = interpolate.interp1d(x, y, kind="cubic")
f_cubic(3.5)                        # interpolated value

xnew = np.linspace(0, 10, 200)
ynew = f_cubic(xnew)                # smooth curve through the points

# Smoothing spline (does not pass exactly through noisy points)
spline = interpolate.UnivariateSpline(x, y, s=0.5)   # s controls smoothness
```

```
   linear:  connect points with straight lines (fast, may look jagged)
   cubic:   smooth curve through all points (can overshoot)
   spline:  smooth + optional noise smoothing via s parameter
```

## 6. scipy.integrate

```python
from scipy import integrate

# Definite integral of a function
result, error = integrate.quad(lambda x: np.exp(-x**2), 0, np.inf)
# result ~ 0.8862 (half of sqrt(pi))

# Integrate sampled data (no analytic function)
y = np.sin(np.linspace(0, np.pi, 100))
integrate.trapezoid(y, dx=np.pi/99)     # trapezoidal rule
integrate.simpson(y, dx=np.pi/99)       # Simpson's rule (more accurate)

# Solve an ODE  dy/dt = f(t, y)
def decay(t, y):
    return -0.5 * y
sol = integrate.solve_ivp(decay, t_span=[0, 10], y0=[1.0],
                          t_eval=np.linspace(0, 10, 100))
sol.t, sol.y      # times and solution values
```

`quad` for analytic functions, `trapezoid`/`simpson` for sampled arrays,
`solve_ivp` for ordinary differential equations.

## 7. scipy.linalg (beyond numpy.linalg)

```python
from scipy import linalg

linalg.inv(A); linalg.det(A); linalg.solve(A, b)      # like numpy but more options
linalg.lu(A)                     # LU decomposition
linalg.cholesky(A)               # for symmetric positive-definite
linalg.qr(A); linalg.svd(A)
linalg.eig(A); linalg.eigh(A)    # eigh for symmetric/Hermitian (faster, stable)
linalg.expm(A)                   # matrix exponential
linalg.norm(A)
linalg.lstsq(A, b)               # least-squares solution to overdetermined system
```

`scipy.linalg` is generally preferred over `numpy.linalg` for serious numerical
work: it always uses optimized LAPACK, offers more decompositions, and has
specialized fast paths (e.g., `eigh` for symmetric matrices).

## 8. scipy.signal (brief)

```python
from scipy import signal

# Filtering
b, a = signal.butter(4, 0.2)                 # design a low-pass Butterworth filter
filtered = signal.filtfilt(b, a, noisy)      # zero-phase filtering

# Convolution / correlation
signal.convolve(x, kernel, mode="same")
signal.correlate(x, y)

# Peak detection
peaks, props = signal.find_peaks(y, height=0.5, distance=10)

# Spectral analysis
freqs, power = signal.periodogram(x, fs=1000)   # power spectral density
signal.spectrogram(x, fs=1000)                  # time-frequency
```

## 9. scipy.sparse (memory for mostly-zero matrices)

```python
from scipy import sparse

# A 10000x10000 dense float64 matrix = 800 MB; sparse stores only nonzeros.
dense = np.eye(1000)
csr = sparse.csr_matrix(dense)      # Compressed Sparse Row (fast row ops, matmul)
csr.nnz                             # number of stored (nonzero) elements
csr @ vector                        # sparse matrix-vector product
sparse.csc_matrix(...)              # column format (fast column ops)
sparse.eye(1000); sparse.diags([...])
```

```
   Use sparse when the matrix is mostly zeros (e.g., one-hot encodings,
   term-document matrices, graph adjacency). scikit-learn's text vectorizers
   return sparse matrices for exactly this reason.
   Formats: CSR (row ops/matmul), CSC (column ops), COO (fast construction).
```

## 10. scipy.spatial (distances and neighbors)

```python
from scipy.spatial import distance, KDTree

distance.euclidean([0, 0], [3, 4])          # 5.0
distance.cdist(A, B, metric="euclidean")    # all pairwise distances (m x n)
distance.pdist(A)                            # condensed pairwise within A

tree = KDTree(points)                        # spatial index for fast neighbor search
dist, idx = tree.query([0, 0], k=3)          # 3 nearest neighbors
tree.query_ball_point([0, 0], r=1.0)         # all points within radius
```

`KDTree` turns O(n) nearest-neighbor scans into O(log n) queries, important for
clustering, k-NN, and geospatial work.

## Exercises

1. For a normal(100, 15): find P(X < 90), the 90th percentile, and draw 1000
   samples; overlay the theoretical pdf on a histogram of the samples.
2. Run a two-sample t-test on two groups and report the t-statistic, p-value, and
   the difference in means (effect size).
3. Fit `a*exp(-b*x)+c` to noisy data with `curve_fit`; report parameter estimates
   and their standard errors from the covariance matrix.
4. Numerically integrate a Gaussian from -inf to inf with `quad` and compare to the
   analytic value.
5. Build a sparse identity-like matrix, confirm its memory footprint is far below
   the dense equivalent, and multiply it by a vector.

Proceed to `14_sklearn.md`.
