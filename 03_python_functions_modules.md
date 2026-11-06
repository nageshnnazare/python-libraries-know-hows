# 03 - Functions, Scope, Modules, Packages

Functions are the primary unit of reuse. Scope rules and the module system are
where many subtle bugs live. This file also covers lambdas and the functional
tools you will use with pandas.

## 1. Defining functions

```python
def add(a, b):
    """Return the sum of a and b."""
    return a + b

result = add(2, 3)        # 5
```

A function with no explicit `return` returns `None`. `return` immediately exits.

```python
def classify(n):
    if n < 0:
        return "negative"
    if n == 0:
        return "zero"
    return "positive"     # no need for else after a return
```

## 2. Parameters and arguments

### Positional and keyword arguments

```python
def power(base, exponent):
    return base ** exponent

power(2, 3)                    # positional -> 8
power(base=2, exponent=3)      # keyword (order-independent)
power(2, exponent=3)           # mix: positional first, then keyword
# power(base=2, 3)             # SyntaxError: positional after keyword
```

### Default parameter values

```python
def greet(name, greeting="Hello"):
    return f"{greeting}, {name}"

greet("Ada")                   # 'Hello, Ada'
greet("Ada", "Hi")             # 'Hi, Ada'
greet("Ada", greeting="Hey")   # 'Hey, Ada'
```

Defaults are evaluated once, at definition time. This causes the single most
famous Python gotcha:

```python
# WRONG: the list is created ONCE and shared across all calls
def append_bad(item, target=[]):
    target.append(item)
    return target

append_bad(1)     # [1]
append_bad(2)     # [1, 2]  <- surprise! same list reused

# RIGHT: use None as sentinel, create fresh inside
def append_good(item, target=None):
    if target is None:
        target = []
    target.append(item)
    return target
```

```
   Mutable default trap:
   def f(x, acc=[]) ─── acc bound to ONE list object at def time
                        every call without acc reuses & mutates it
   Fix: default to None, allocate inside the body.
```

Never use a mutable object (`[]`, `{}`, `set()`) as a default value.

### *args and **kwargs (variadic functions)

```python
def total(*args):              # args is a tuple of extra positionals
    return sum(args)
total(1, 2, 3, 4)              # 10

def config(**kwargs):          # kwargs is a dict of extra keywords
    for k, v in kwargs.items():
        print(k, v)
config(debug=True, level=3)

def flexible(required, *args, **kwargs):
    print(required, args, kwargs)
flexible("a", 1, 2, x=10)      # a (1, 2) {'x': 10}
```

Unpacking at the call site (the reverse operation):

```python
nums = [1, 2, 3]
total(*nums)                   # spreads list into positional args
opts = {"debug": True}
config(**opts)                 # spreads dict into keyword args
```

```
   DEFINITION side:  *args   collects extra positionals into a tuple
                     **kwargs collects extra keywords into a dict
   CALL side:        *seq    spreads a sequence into positional args
                     **map   spreads a mapping into keyword args
```

### Positional-only and keyword-only parameters (3.8+)

```python
def f(pos_only, /, normal, *, kw_only):
    ...
# everything before /  must be positional
# everything after  *  must be keyword
f(1, 2, kw_only=3)          # ok
f(1, normal=2, kw_only=3)   # ok
# f(pos_only=1, ...)        # error: pos_only cannot be a keyword
```

Use keyword-only for boolean flags and options to force readable call sites:
`resize(img, keep_aspect=True)` reads better than `resize(img, True)`.

## 3. Scope: LEGB rule

Names are resolved in this order:

```
   L ─ Local      (inside the current function)
   E ─ Enclosing  (any outer function's scope)
   G ─ Global     (module level)
   B ─ Built-in   (len, print, range, ...)

   Lookup goes L -> E -> G -> B, stopping at the first match.
```

```python
x = "global"

def outer():
    x = "enclosing"
    def inner():
        x = "local"
        print(x)       # local
    inner()
    print(x)           # enclosing

outer()
print(x)               # global
```

### Modifying outer scope: global and nonlocal

```python
counter = 0
def increment():
    global counter      # rebind the module-level name
    counter += 1

def make_counter():
    count = 0
    def step():
        nonlocal count  # rebind the enclosing function's name
        count += 1
        return count
    return step
```

Without `global`/`nonlocal`, assigning to a name creates a *new local* variable,
shadowing the outer one. Reading (not assigning) outer variables needs no
declaration. Overusing `global` is a code smell; prefer returning values.

## 4. Closures

A closure is a nested function that captures variables from its enclosing scope,
keeping them alive after the outer function returns.

```python
def multiplier(factor):
    def multiply(n):
        return n * factor    # captures 'factor'
    return multiply

double = multiplier(2)
triple = multiplier(3)
double(10)     # 20
triple(10)     # 30
```

```
   multiplier(2) returns 'multiply' which REMEMBERS factor=2
   ┌──────────────────────────────┐
   │ closure: multiply            │
   │   free var factor = 2 ───────┼──> captured from enclosing scope
   └──────────────────────────────┘
```

Late-binding gotcha in loops:

```python
funcs = [lambda: i for i in range(3)]
[f() for f in funcs]          # -> [2, 2, 2]  (all see final i)
# Fix: bind per-iteration via a default argument
funcs = [lambda i=i: i for i in range(3)]
[f() for f in funcs]          # -> [0, 1, 2]
```

## 5. Lambda functions

Anonymous, single-expression functions. Use for short callbacks; do not assign
them to names (use `def` instead for readability).

```python
square = lambda x: x**2        # works but prefer def for named functions
(lambda x, y: x + y)(2, 3)     # 5

# Idiomatic use: as a key/callback argument
sorted(pairs, key=lambda p: p[1])          # sort by 2nd element
sorted(words, key=lambda w: w.lower())     # case-insensitive sort
max(items, key=lambda x: x.score)
```

Lambdas can hold only one expression (no statements, no assignments). If you need
more, use `def`.

## 6. Functional tools: map, filter, reduce

```python
nums = [1, 2, 3, 4]
list(map(lambda x: x**2, nums))            # [1, 4, 9, 16]
list(filter(lambda x: x % 2 == 0, nums))   # [2, 4]

from functools import reduce
reduce(lambda a, b: a + b, nums)           # 10 (running fold)
reduce(lambda a, b: a + b, nums, 100)      # 110 (with initial value)
```

`map`/`filter` are lazy (return iterators). In practice, comprehensions are more
readable and usually preferred:

```python
[x**2 for x in nums]              # clearer than map(lambda...)
[x for x in nums if x % 2 == 0]   # clearer than filter(lambda...)
```

Useful `functools` helpers:

```python
from functools import partial, lru_cache, reduce

# partial: pre-fill some arguments
def power(base, exp): return base ** exp
square = partial(power, exp=2)
square(5)          # 25

# lru_cache: memoize expensive/pure functions
@lru_cache(maxsize=None)
def fib(n):
    return n if n < 2 else fib(n-1) + fib(n-2)
fib(100)           # fast; results cached
```

## 7. Type hints (annotations)

Hints document intent and enable static checkers (mypy, pyright). They do NOT
enforce types at runtime.

```python
def greet(name: str, times: int = 1) -> str:
    return f"Hello {name}! " * times

from typing import Optional, Union

def parse(text: str) -> Optional[int]:   # may return int or None
    return int(text) if text.isdigit() else None

# Modern syntax (3.10+) uses | instead of Union/Optional:
def parse2(text: str) -> int | None:
    ...

# Collections
def totals(data: list[float]) -> dict[str, float]:
    ...
def coords() -> tuple[float, float]:
    ...

from collections.abc import Callable, Iterable
def apply(fn: Callable[[int], int], xs: Iterable[int]) -> list[int]:
    return [fn(x) for x in xs]
```

Hints are worth adopting even solo: they catch bugs and make functions
self-documenting. Trade-off: verbosity. In data-heavy code, hint public function
signatures at minimum.

## 8. Docstrings and help

```python
def normalize(x, lo, hi):
    """Scale x to [0, 1] given range [lo, hi].

    Args:
        x: value to scale.
        lo: minimum of the source range.
        hi: maximum of the source range (must be > lo).
    Returns:
        float in [0, 1].
    Raises:
        ZeroDivisionError: if hi == lo.
    Example:
        >>> normalize(5, 0, 10)
        0.5
    """
    return (x - lo) / (hi - lo)
```

The `>>> ` examples are testable with the `doctest` module.

## 9. Modules

A module is any `.py` file. It groups related code and creates a namespace.

```python
# ---- mymath.py ----
PI = 3.14159
def area(r):
    return PI * r * r
```

```python
# ---- main.py ----
import mymath
mymath.area(2)                 # qualified access

from mymath import area, PI    # import specific names
area(2)

from mymath import area as circle_area   # alias
import mymath as mm            # module alias

# Avoid: from mymath import *  (pollutes namespace, hides origins)
```

### The import search path

```
   Python looks for a module in sys.path, in order:
   1. the directory of the running script (or cwd in REPL)
   2. directories in the PYTHONPATH env var
   3. installation-dependent site-packages (installed packages)
```

```python
import sys
sys.path            # inspect the search path
```

### `if __name__ == "__main__"`

When a file is run directly, its `__name__` is `"__main__"`. When imported, it is
the module name. This guard separates "run as script" from "import as library".

```python
def main():
    print("running")

if __name__ == "__main__":
    main()          # runs only when executed directly, not on import
```

```
   $ python mymath.py     -> __name__ == "__main__"  -> main() runs
   >>> import mymath      -> __name__ == "mymath"    -> main() skipped
```

Always guard script entry points this way so the module can be safely imported.

## 10. Packages

A package is a directory of modules. Historically it needed an `__init__.py`;
namespace packages (3.3+) can omit it, but including it is clearer.

```
   myproject/
   ├── __init__.py          <- makes 'myproject' a package
   ├── data/
   │   ├── __init__.py
   │   ├── load.py
   │   └── clean.py
   └── models/
       ├── __init__.py
       └── train.py
```

```python
from myproject.data.load import read_csv
from myproject.models import train
import myproject.data.clean as clean
```

Relative imports (inside a package):

```python
# inside myproject/models/train.py
from ..data.load import read_csv    # .. = parent package
from .helpers import util           # .  = current package
```

`__init__.py` runs on package import; use it to expose a clean public API:

```python
# myproject/__init__.py
from .data.load import read_csv
from .models.train import fit
__all__ = ["read_csv", "fit"]       # controls 'from myproject import *'
```

## 11. Modules are cached and run once

The first `import` executes the module top-to-bottom and caches it in
`sys.modules`. Subsequent imports reuse the cache (they do not re-run the file).
Put executable side effects behind the `__main__` guard so importing is cheap and
side-effect-free.

## Exercises

1. Write `def stats(*nums)` returning `(min, max, mean)`; handle the empty case.
2. Write a `make_adder(n)` closure returning a function that adds `n`.
3. Fix this: `def collect(x, into=[]): into.append(x); return into`.
4. Split a small script into a `utils.py` module and a `main.py` that imports it,
   using the `__main__` guard.
5. Use `functools.lru_cache` to memoize a recursive function and compare speed.

Proceed to `04_python_oop.md`.
