# 05 - Advanced Python

Error handling, iterators and generators, decorators, context managers, advanced
typing, concurrency, and a few performance notes. These are the tools that
separate intermediate from expert Python, and several (generators, context
managers) appear constantly in data pipelines.

## 1. Exceptions and error handling

```python
try:
    result = 10 / x
except ZeroDivisionError:
    result = float("inf")
except (TypeError, ValueError) as e:     # catch multiple, capture the object
    print(f"bad input: {e}")
else:
    print("no error occurred")           # runs only if no exception
finally:
    print("always runs")                 # cleanup, runs no matter what
```

```
   try:      attempt risky code
     │
     ├── exception raised ──> matching except block runs
     │                           │
     └── no exception ────────> else block runs
                                 │
   finally: ALWAYS runs (cleanup), whether or not an exception occurred
```

### Raising and re-raising

```python
def set_age(age):
    if age < 0:
        raise ValueError(f"age cannot be negative: {age}")
    return age

try:
    risky()
except OSError as e:
    log(e)
    raise                       # re-raise the same exception, preserving traceback

# Exception chaining (show root cause):
try:
    parse()
except ValueError as e:
    raise RuntimeError("config invalid") from e
```

### Custom exceptions

```python
class ValidationError(Exception):
    """Raised when input validation fails."""

class RangeError(ValidationError):     # hierarchy: catch by base or specific
    def __init__(self, value, lo, hi):
        super().__init__(f"{value} not in [{lo}, {hi}]")
        self.value = value

try:
    raise RangeError(15, 0, 10)
except ValidationError as e:           # base class catches subclasses
    print(e)
```

### The exception hierarchy (know the common ones)

```
   BaseException
    └── Exception              <- catch this or subclasses, NOT BaseException
         ├── ArithmeticError
         │    └── ZeroDivisionError
         ├── LookupError
         │    ├── IndexError    (list[999])
         │    └── KeyError      (dict["missing"])
         ├── ValueError         (int("abc"))
         ├── TypeError          (1 + "a")
         ├── FileNotFoundError  (open("missing"))
         ├── AttributeError     (obj.no_such_attr)
         └── StopIteration
```

Best practices:

- Catch the most specific exception you can handle; avoid bare `except:`.
- Never silently swallow errors (`except: pass`) unless you truly mean to.
- Use `try/except` for exceptional cases, not normal control flow you can check
  cheaply first ("easier to ask forgiveness than permission" is idiomatic, but do
  not hide real bugs).

```python
# EAFP (Pythonic): try and handle failure
try:
    value = d[key]
except KeyError:
    value = default
# vs LBYL (look before you leap):
value = d[key] if key in d else default    # or just d.get(key, default)
```

## 2. Iterators and the iteration protocol

An iterable produces an iterator via `__iter__`; an iterator yields values via
`__next__` and raises `StopIteration` when exhausted.

```python
nums = [1, 2, 3]
it = iter(nums)          # get an iterator
next(it)                 # 1
next(it)                 # 2
next(it)                 # 3
# next(it)               # raises StopIteration
```

```
   for x in iterable:  ─── desugars to ───
       it = iter(iterable)
       while True:
           try:    x = next(it)
           except StopIteration: break
           <loop body>
```

Custom iterator:

```python
class CountDown:
    def __init__(self, start):
        self.n = start
    def __iter__(self):
        return self
    def __next__(self):
        if self.n <= 0:
            raise StopIteration
        self.n -= 1
        return self.n + 1

list(CountDown(3))       # [3, 2, 1]
```

## 3. Generators (essential for data pipelines)

Generators are the easy way to build iterators. A function with `yield` returns a
generator that produces values lazily, one at a time, suspending state between
calls. This gives near-constant memory for arbitrarily large streams.

```python
def countdown(n):
    while n > 0:
        yield n          # pause here, return n, resume on next()
        n -= 1

for x in countdown(3):
    print(x)             # 3, 2, 1

gen = countdown(3)
next(gen)                # 3
```

```
   Regular function          Generator function
   ─────────────────         ──────────────────
   builds entire result      yields values one at a time
   returns once              suspends/resumes at each yield
   memory ~ O(n)             memory ~ O(1)
```

### Why generators matter for data science

```python
# Read a huge file line-by-line without loading it all into memory:
def read_large(path):
    with open(path) as f:
        for line in f:
            yield line.strip()

# Chain lazy transformations (nothing computed until consumed):
def pipeline(path):
    lines = read_large(path)
    numbers = (int(x) for x in lines if x.isdigit())
    squared = (n * n for n in numbers)
    return squared

total = sum(pipeline("data.txt"))   # streams the whole file, tiny memory
```

```
   file ──> read_large ──> filter digits ──> square ──> sum
   (each stage pulls ONE item at a time; no intermediate lists)
```

### yield from (delegate to a sub-generator)

```python
def chain(*iterables):
    for it in iterables:
        yield from it            # flatten multiple iterables lazily
list(chain([1, 2], [3, 4]))      # [1, 2, 3, 4]
```

Generators are single-use: once exhausted, you must create a new one.

## 4. Decorators

A decorator is a callable that takes a function and returns a modified function.
Used for cross-cutting concerns: logging, timing, caching, access control,
retries. You will see `@decorator` syntax everywhere.

```python
import functools, time

def timer(func):
    @functools.wraps(func)          # preserve func's name/docstring
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"{func.__name__} took {elapsed:.4f}s")
        return result
    return wrapper

@timer                              # equivalent to: slow = timer(slow)
def slow(n):
    return sum(range(n))

slow(1_000_000)                     # prints timing, returns result
```

```
   @timer
   def slow(...): ...

   is exactly:

   def slow(...): ...
   slow = timer(slow)      <- slow now points to wrapper, which wraps the original
```

`functools.wraps` copies metadata (`__name__`, `__doc__`) from the original to the
wrapper; without it, introspection and debugging break.

### Decorator with arguments (a decorator factory)

```python
def retry(times=3):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(times):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == times - 1:
                        raise
                    print(f"retry {attempt+1} after {e}")
        return wrapper
    return decorator

@retry(times=5)
def flaky():
    ...
```

```
   retry(times=5)  returns  decorator
   decorator(flaky) returns wrapper
   -> three nested layers: factory -> decorator -> wrapper
```

Common built-in decorators: `@property`, `@staticmethod`, `@classmethod`
(file 04), `@functools.lru_cache`, `@functools.cache` (3.9+), `@dataclass`.

## 5. Context managers (the `with` statement)

Guarantee setup and teardown (acquire/release) even if an exception occurs. The
canonical example is file handling.

```python
with open("data.txt") as f:      # __enter__ opens; __exit__ closes
    data = f.read()
# file is closed here automatically, even if read() raised
```

```
   with CM as x:          x = CM.__enter__()
       body               try: body
                          finally: CM.__exit__(exc_type, exc, tb)
```

Multiple context managers:

```python
with open("in.txt") as fin, open("out.txt", "w") as fout:
    fout.write(fin.read())
```

### Writing your own: class-based

```python
class Timer:
    def __enter__(self):
        self.start = time.perf_counter()
        return self                      # value bound to 'as'
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.elapsed = time.perf_counter() - self.start
        print(f"elapsed: {self.elapsed:.4f}s")
        return False                     # False = do not suppress exceptions

with Timer():
    do_work()
```

### Writing your own: contextlib (simpler)

```python
from contextlib import contextmanager

@contextmanager
def timer():
    start = time.perf_counter()
    try:
        yield                            # everything before yield = __enter__
    finally:
        print(f"{time.perf_counter() - start:.4f}s")  # after yield = __exit__

with timer():
    do_work()
```

`__exit__` receiving exception info lets a context manager clean up (close files,
release locks, roll back transactions) reliably.

## 6. Advanced typing

```python
from typing import TypeVar, Generic, Protocol, Literal, TypedDict, overload
from collections.abc import Sequence, Callable

# Generics: preserve the element type through a function
T = TypeVar("T")
def first(items: Sequence[T]) -> T:
    return items[0]

# Protocol: structural typing ("duck typing" checked statically)
class SupportsClose(Protocol):
    def close(self) -> None: ...
def shutdown(resource: SupportsClose) -> None:
    resource.close()               # any object with close() qualifies

# Literal: restrict to specific values
def set_mode(mode: Literal["r", "w", "a"]) -> None: ...

# TypedDict: type a dict's shape
class Movie(TypedDict):
    title: str
    year: int

# Type alias
Matrix = list[list[float]]
```

Run a static checker to benefit: `pip install mypy && mypy yourfile.py`. Hints are
optional and ignored at runtime, but catch a large class of bugs before you run.

## 7. Concurrency: choosing a model

```
   ┌────────────────┬──────────────────────┬───────────────────────────┐
   │ Model          │ Best for             │ Why                       │
   ├────────────────┼──────────────────────┼───────────────────────────┤
   │ threading      │ I/O-bound            │ threads wait on I/O, GIL  │
   │                │ (network, disk)      │ released during I/O       │
   │ multiprocessing│ CPU-bound            │ separate processes bypass │
   │                │ (heavy computation)  │ the GIL, use many cores   │
   │ asyncio        │ many concurrent I/O  │ single thread, cooperative│
   │                │ (thousands of conns) │ scheduling, no thread cost│
   └────────────────┴──────────────────────┴───────────────────────────┘
```

The GIL (Global Interpreter Lock) in CPython allows only one thread to execute
Python bytecode at a time. So threads do NOT speed up pure-Python CPU work; use
processes for that. (Note: NumPy/pandas release the GIL in their C code, so
threaded numeric work can still parallelize; and Python 3.13+ has an experimental
free-threaded build.)

```python
# I/O-bound: thread pool
from concurrent.futures import ThreadPoolExecutor
with ThreadPoolExecutor(max_workers=8) as ex:
    results = list(ex.map(download, urls))

# CPU-bound: process pool
from concurrent.futures import ProcessPoolExecutor
with ProcessPoolExecutor() as ex:
    results = list(ex.map(heavy_compute, chunks))
```

```python
# asyncio: cooperative concurrency for I/O
import asyncio
async def fetch(url):
    await asyncio.sleep(1)          # 'await' yields control while waiting
    return url

async def main():
    results = await asyncio.gather(*(fetch(u) for u in urls))  # concurrent
    return results

asyncio.run(main())
```

For data science, the practical rule: let NumPy/pandas/scikit-learn handle
parallelism internally (many have `n_jobs` params); reach for `ProcessPoolExecutor`
for embarrassingly parallel custom CPU work; use threads/asyncio for fetching many
files or API calls.

## 8. Performance notes

```python
# Measure, do not guess:
import timeit
timeit.timeit("sum(range(100))", number=100000)

# In IPython/Jupyter:
# %timeit sum(range(100))
# %prun my_function()        # profile function calls
# %memit my_function()       # memory (needs memory_profiler)
```

General ordering from slow to fast for numeric work:

```
   pure Python loop  <  list comprehension  <  built-in (sum/map)  <  NumPy vectorized
   (slowest)                                                          (fastest)
```

The single biggest lever in data science is *vectorization*: replace Python-level
loops with NumPy/pandas operations that run in C (files 06-10).

Other quick wins:

- Use sets/dicts for membership instead of lists (O(1) vs O(n)).
- Use generators to avoid building large intermediate lists.
- Use `str.join` to build strings, not `+=` in a loop (which is O(n^2)).
- Cache pure expensive functions with `functools.cache`.
- Prefer local variable references in hot loops (attribute lookups cost time).

## 9. Miscellaneous expert idioms

```python
# Walrus operator := (assign within an expression, 3.8+)
while (line := input()) != "quit":
    print(line)
if (n := len(data)) > 100:
    print(f"large: {n}")

# Unpacking in function calls and merges
combined = {**defaults, **overrides}
merged = [*list_a, *list_b]

# Ternary and chained comparisons in comprehensions
grades = ["pass" if s >= 60 else "fail" for s in scores]

# any / all for boolean aggregation
all(x > 0 for x in nums)      # True if every element positive
any(x < 0 for x in nums)      # True if at least one negative

# Enumerate with unpacking, sorted with itemgetter
from operator import itemgetter, attrgetter
sorted(records, key=itemgetter("age"))   # faster than lambda for dicts
```

## Exercises

1. Write a generator `windows(seq, n)` yielding successive n-length tuples (a
   sliding window). Verify it uses O(1) extra memory.
2. Write a `@memoize` decorator using a dict; compare to `functools.cache`.
3. Write a context manager `suppress_and_log(*exceptions)` that logs and swallows
   the given exception types (compare to `contextlib.suppress`).
4. Convert a list-building function to a generator and measure the memory
   difference with a large input.
5. Decide (and justify) threading vs multiprocessing vs asyncio for: (a) resizing
   10k images, (b) fetching 1k URLs, (c) summing a huge list of numbers.

Proceed to `06_numpy_fundamentals.md`.
