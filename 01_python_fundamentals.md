# 01 - Python Fundamentals

Everything here is core Python; no external libraries yet. If you already know
another language, skim for the Python-specific behaviors (indentation, dynamic
typing, truthiness, integer size, `is` vs `==`).

## 1. The execution model

Python is interpreted and dynamically typed. Source (`.py`) is compiled to
bytecode (`.pyc`) and run on the Python Virtual Machine (PVM).

```
   source.py ──compile──> bytecode (.pyc) ──execute──> PVM ──> OS
     (text)               (instructions)              (loop)
```

Dynamic typing means variables are just names bound to objects; the *object*
carries the type, not the variable.

```python
x = 5          # name x -> int object 5
x = "hello"    # name x -> str object "hello"  (perfectly legal)
```

```
   Names (variables)          Objects (in memory)
   ┌───────┐                  ┌──────────────┐
   │   x   │ ───────────────> │ int   5      │
   └───────┘                  └──────────────┘
   after reassignment:
   ┌───────┐                  ┌──────────────┐
   │   x   │ ───────────────> │ str "hello"  │
   └───────┘                  └──────────────┘
                              (the int 5 is garbage-collected if unreferenced)
```

## 2. Syntax basics: indentation is structure

Python uses indentation (not braces) to define blocks. The standard is 4 spaces.
Mixing tabs and spaces is an error.

```python
def greet(name):
    if name:                     # block starts with ':'
        print(f"Hello, {name}")  # 4-space indent = inside the if
    else:
        print("Hello, stranger")
    print("done")                # back to function level
```

Statements end at newline. To continue a long line, use parentheses (preferred)
or a backslash (avoid):

```python
total = (1 + 2 + 3 +
         4 + 5)          # implicit continuation inside ()
total = 1 + 2 + 3 + \
        4 + 5            # explicit continuation (fragile, avoid)
```

## 3. Variables, naming, assignment

```python
count = 0                     # snake_case for variables/functions
MAX_RETRIES = 3               # UPPER_CASE for constants (convention only)
first_name, last_name = "Ada", "Lovelace"   # tuple unpacking
a = b = c = 0                 # chained assignment
x, y = y, x                   # swap without a temp variable
first, *rest = [1, 2, 3, 4]   # star unpacking: first=1, rest=[2,3,4]
```

Rules: names start with a letter or underscore, contain letters/digits/underscores,
are case-sensitive, and cannot be reserved keywords (`for`, `class`, `def`, ...).

## 4. Numeric types

```python
i = 42            # int  - arbitrary precision (no overflow)
f = 3.14          # float - IEEE 754 double (64-bit)
c = 2 + 3j        # complex
b = 0b1010        # binary literal -> 10
h = 0xff          # hex literal -> 255
big = 1_000_000   # underscores as digit separators (readability)
```

Python ints have no fixed size; they grow as needed:

```python
2 ** 100          # -> 1267650600228229401496703205376  (no overflow)
```

Float precision is the classic IEEE 754 caveat:

```python
0.1 + 0.2 == 0.3          # -> False
0.1 + 0.2                 # -> 0.30000000000000004
# For exact decimal arithmetic (money), use the decimal module:
from decimal import Decimal
Decimal("0.1") + Decimal("0.2") == Decimal("0.3")   # -> True
```

Operators:

```python
7 / 2      # -> 3.5    true division (always float)
7 // 2     # -> 3      floor division
7 % 2      # -> 1      modulo
7 ** 2     # -> 49     exponent
divmod(7, 2)  # -> (3, 1)  quotient and remainder together
```

## 5. Strings

Strings are immutable sequences of Unicode characters.

```python
s = "double"
s = 'single'                # equivalent
s = """multi
line"""                     # triple quotes span lines
s = 'He said "hi"'          # mix quotes to avoid escaping
```

### f-strings (formatted string literals, the modern default)

```python
name, score = "Ada", 95.678
f"{name} scored {score}"          # -> 'Ada scored 95.678'
f"{score:.1f}"                    # -> '95.7'   (1 decimal place)
f"{score:8.2f}"                   # -> '   95.68' (width 8, 2 decimals)
f"{1000000:,}"                    # -> '1,000,000' (thousands separator)
f"{0.25:.1%}"                     # -> '25.0%' (percent)
f"{255:#x}"                       # -> '0xff' (hex with prefix)
f"{name=}"                        # -> "name='Ada'"  (debug =, Python 3.8+)
```

### Common string methods

```python
s = "  Hello, World  "
s.strip()              # -> 'Hello, World'   (remove surrounding whitespace)
s.lower()              # -> '  hello, world  '
s.upper()
s.replace("o", "0")    # -> '  Hell0, W0rld  '
"a,b,c".split(",")     # -> ['a', 'b', 'c']
"-".join(["a","b"])    # -> 'a-b'
"data.csv".endswith(".csv")   # -> True
"abc".startswith("ab")        # -> True
"Hello".find("l")             # -> 2 (index of first match, -1 if absent)
"hello world".title()         # -> 'Hello World'
"42".zfill(5)                 # -> '00042'
"a1b2".isalnum()              # -> True
```

### Indexing and slicing (applies to all sequences)

```python
s = "PYTHON"
#    P  Y  T  H  O  N
#    0  1  2  3  4  5     (positive indices)
#   -6 -5 -4 -3 -2 -1     (negative indices)

s[0]      # -> 'P'
s[-1]     # -> 'N'
s[1:4]    # -> 'YTH'   (start inclusive, stop exclusive)
s[:3]     # -> 'PYT'   (from start)
s[3:]     # -> 'HON'   (to end)
s[::2]    # -> 'PTO'   (every 2nd char)
s[::-1]   # -> 'NOHTYP' (reverse)
```

Slice syntax is `[start:stop:step]`. This same syntax works on lists, tuples, and
NumPy arrays, so learn it well.

## 6. Booleans and truthiness

```python
True, False           # the two bool values (subclass of int: True == 1)
```

Every object has a truth value. "Falsy" values:

```python
bool(0)        # False
bool(0.0)      # False
bool("")       # False (empty string)
bool([])       # False (empty list)
bool({})       # False (empty dict)
bool(())       # False (empty tuple)
bool(None)     # False
bool(set())    # False
# Everything else is truthy.
```

This enables idiomatic checks:

```python
items = []
if items:            # preferred over: if len(items) > 0
    process(items)
if not items:        # preferred over: if len(items) == 0
    print("empty")
```

## 7. Comparison, logical, identity operators

```python
# Comparison (return bool)
1 < 2, 2 <= 2, 3 == 3, 3 != 4, 5 > 1, 5 >= 5

# Chained comparison (Pythonic; each term evaluated once)
x = 5
1 < x < 10           # -> True   (means 1 < x and x < 10)

# Logical operators (short-circuit)
True and False       # -> False
True or False        # -> True
not True             # -> False

# and/or return an operand, not necessarily a bool:
"a" or "b"           # -> 'a'  (first truthy)
"" or "default"      # -> 'default'
0 and 5              # -> 0    (first falsy)

# Identity vs equality
a = [1, 2]; b = [1, 2]
a == b               # -> True   (same value)
a is b               # -> False  (different objects in memory)
a is not None        # correct way to test for None
```

`==` compares values; `is` compares identity (same object). Use `is` only for
`None`, `True`, `False`, and sentinel singletons.

```
   ==  asks: "do these have the same value?"
   is  asks: "are these literally the same object?"
```

CPython caches small ints (-5..256) and some strings, so `is` may *accidentally*
work for small numbers. Never rely on it:

```python
256 is 256           # -> True  (cached)
257 is 257           # -> may be False; do not depend on this
```

## 8. None

`None` is the singleton representing "no value". Functions with no `return` return
`None`.

```python
result = None
if result is None:
    result = compute()
```

## 9. Type conversion (casting)

```python
int("42")        # -> 42
int(3.9)         # -> 3   (truncates toward zero, not rounds)
float("3.14")    # -> 3.14
str(42)          # -> '42'
bool(0)          # -> False
list("abc")      # -> ['a', 'b', 'c']
int("ff", 16)    # -> 255  (parse hex)
round(3.14159, 2)  # -> 3.14
round(2.5)       # -> 2  (banker's rounding: rounds to even!)
round(3.5)       # -> 4
```

`round()` uses banker's rounding (round-half-to-even) to reduce statistical bias.
This surprises people; note it.

## 10. Control flow: conditionals

```python
score = 85
if score >= 90:
    grade = "A"
elif score >= 80:            # elif = else if
    grade = "B"
elif score >= 70:
    grade = "C"
else:
    grade = "F"
```

Ternary (conditional expression):

```python
status = "pass" if score >= 60 else "fail"
```

`match` statement (structural pattern matching, Python 3.10+):

```python
def http_message(status):
    match status:
        case 200:
            return "OK"
        case 404:
            return "Not Found"
        case 500 | 502 | 503:        # multiple values with |
            return "Server Error"
        case int() if status >= 400:  # guard condition
            return "Client/Other Error"
        case _:                       # wildcard (default)
            return "Unknown"
```

## 11. Control flow: loops

### for loops iterate over sequences

```python
for i in range(5):           # 0,1,2,3,4
    print(i)

for i in range(2, 10, 2):    # start, stop, step -> 2,4,6,8
    print(i)

for char in "abc":
    print(char)

for item in [10, 20, 30]:
    print(item)
```

`range(stop)`, `range(start, stop)`, `range(start, stop, step)`. `range` is lazy;
it does not build a list.

### enumerate: index and value together

```python
for index, value in enumerate(["a", "b", "c"]):
    print(index, value)       # 0 a / 1 b / 2 c

for index, value in enumerate(["a", "b"], start=1):
    print(index, value)       # 1 a / 2 b
```

Prefer `enumerate` over `range(len(x))`; it is clearer and less error-prone.

### zip: iterate multiple sequences in parallel

```python
names = ["Ada", "Alan"]
ages = [36, 41]
for name, age in zip(names, ages):
    print(name, age)          # Ada 36 / Alan 41
# zip stops at the shortest input. Use zip(..., strict=True) (3.10+)
# to raise if lengths differ.
```

### while loops

```python
n = 10
while n > 0:
    print(n)
    n -= 1
```

### Loop control: break, continue, else

```python
for x in range(10):
    if x == 5:
        break            # exit loop entirely
    if x % 2 == 0:
        continue         # skip to next iteration
    print(x)             # prints 1, 3

# for/while can have an else: runs if the loop finished WITHOUT break
for x in [1, 3, 5]:
    if x % 2 == 0:
        print("found even")
        break
else:
    print("no even numbers")   # this runs (no break happened)
```

The `for...else` construct is uncommon but useful for search loops: the `else`
means "the search completed without finding anything".

## 12. Input and output

```python
name = input("Enter your name: ")   # always returns a str
age = int(input("Enter age: "))     # cast explicitly

print("a", "b", "c")                # -> a b c   (space separator)
print("a", "b", sep="-")            # -> a-b
print("no newline", end="")         # suppress trailing newline
print("x", "y", sep=", ", end="!\n")
```

## 13. Comments and docstrings

```python
# A single-line comment. Explain WHY, not WHAT.

def area(radius):
    """Return the area of a circle.

    Args:
        radius: non-negative radius in the same unit as the result's sqrt.
    Returns:
        float area = pi * r**2.
    """
    import math
    return math.pi * radius ** 2

help(area)          # prints the docstring
area.__doc__        # access it programmatically
```

Comments start with `#`. Docstrings are the first string literal in a module,
function, or class; tools and `help()` read them.

## 14. The standard library you will actually use

```python
import math
math.sqrt(16)     # 4.0
math.pi, math.e   # constants
math.floor(3.7)   # 3
math.ceil(3.2)    # 4
math.log(100, 10) # 2.0
math.inf, -math.inf, math.nan

import random
random.random()             # float in [0.0, 1.0)
random.randint(1, 6)        # int in [1, 6] inclusive
random.choice(["a","b"])    # random element
random.shuffle(mylist)      # in-place shuffle
random.seed(42)             # reproducibility

import datetime as dt
today = dt.date.today()
now = dt.datetime.now()
delta = dt.timedelta(days=7)
next_week = today + delta

import os
os.getcwd()                 # current directory
os.environ.get("HOME")      # env var (returns None if absent)

from pathlib import Path     # modern path handling (prefer over os.path)
p = Path("data") / "file.csv"    # OS-independent path join
p.exists(), p.suffix, p.stem, p.parent
```

## 15. Common beginner pitfalls

```python
# 1. Integer division vs float division
5 / 2      # 2.5  (not 2)  -- use // for integer result

# 2. Mutable default arguments (covered in file 03) -- do NOT do:
def bad(items=[]):    # the list is shared across calls!
    items.append(1)
    return items

# 3. Comparing floats with ==
0.1 + 0.2 == 0.3      # False -- use math.isclose(a, b)
import math
math.isclose(0.1 + 0.2, 0.3)   # True

# 4. Modifying a list while iterating over it (skips elements)
# 5. Confusing = (assignment) with == (comparison)
```

## Exercises

1. Write a loop that prints FizzBuzz for 1..30 (Fizz on multiples of 3, Buzz on 5,
   FizzBuzz on both).
2. Given `s = "hello world"`, produce `"Dlrow Olleh"` using slicing and methods.
3. Read a number and print whether it is prime.
4. Use `zip` and `enumerate` together to print `"1: Ada & 36"` style lines.

Proceed to `02_python_data_structures.md`.
