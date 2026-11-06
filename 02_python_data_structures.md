# 02 - Python Data Structures

The four built-in collections plus comprehensions. Mastering these is the single
highest-leverage Python skill for data work, because pandas and NumPy build on the
same mental models.

## Overview and when to use each

```
   ┌───────────┬──────────┬──────────┬───────────┬─────────────────────────┐
   │ Type      │ Ordered  │ Mutable  │ Duplicates│ Typical use             │
   ├───────────┼──────────┼──────────┼───────────┼─────────────────────────┤
   │ list      │ yes      │ yes      │ yes       │ ordered sequence        │
   │ tuple     │ yes      │ no       │ yes       │ fixed record, dict key  │
   │ dict      │ yes*     │ yes      │ keys:no   │ key -> value mapping    │
   │ set       │ no       │ yes      │ no        │ membership, dedup       │
   └───────────┴──────────┴──────────┴───────────┴─────────────────────────┘
   * dict preserves insertion order since Python 3.7 (guaranteed).
```

Selection guide:

```
   Need key -> value lookup?  ──yes──> dict
        │no
   Need uniqueness / set math? ──yes──> set
        │no
   Will it change after creation? ──no──> tuple
        │yes
        └──> list
```

## 1. Lists

Ordered, mutable sequences. The workhorse collection.

```python
nums = [3, 1, 4, 1, 5]
mixed = [1, "two", 3.0, [4, 5]]   # can hold any types (usually don't mix)
empty = []
from_iter = list(range(5))        # [0, 1, 2, 3, 4]
```

### Access and slicing (same slice rules as strings)

```python
nums[0]       # 3
nums[-1]      # 5
nums[1:3]     # [1, 4]
nums[::-1]    # reversed copy
```

### Mutating methods (modify in place, return None)

```python
lst = [1, 2, 3]
lst.append(4)          # [1,2,3,4]           add one item to end
lst.extend([5, 6])     # [1,2,3,4,5,6]       add many items
lst.insert(0, 0)       # [0,1,2,3,4,5,6]     insert at index
lst.remove(3)          # remove first value == 3
lst.pop()              # remove & return last item
lst.pop(0)             # remove & return item at index 0
lst.sort()             # sort in place (ascending)
lst.sort(reverse=True) # descending
lst.reverse()          # reverse in place
lst.clear()            # empty the list
```

`.append` vs `.extend` is a common bug source:

```python
a = [1, 2]
a.append([3, 4])   # -> [1, 2, [3, 4]]   (nested!)
a = [1, 2]
a.extend([3, 4])   # -> [1, 2, 3, 4]     (flattened)
```

### Non-mutating operations (return new value)

```python
sorted([3, 1, 2])          # -> [1, 2, 3]  (original unchanged)
sorted(words, key=len)     # sort by a function's result
sorted(data, key=lambda r: r[1], reverse=True)
len(nums)                  # count
sum(nums)                  # total
min(nums), max(nums)
nums.count(1)              # occurrences of 1
nums.index(4)              # first index of value 4 (ValueError if absent)
3 in nums                  # membership test -> bool
[1, 2] + [3, 4]            # concatenation -> [1,2,3,4]
[0] * 3                    # repetition -> [0, 0, 0]
```

### The copy trap (references vs copies)

Assignment binds a new name to the *same* list. Mutating through one name is
visible through the other.

```python
a = [1, 2, 3]
b = a              # b and a point to the SAME list
b.append(4)
a                  # -> [1, 2, 3, 4]   (a changed too!)

# Make a shallow copy:
b = a.copy()       # or a[:] or list(a)
b.append(5)
a                  # unchanged
```

```
   Shallow copy of [1, 2, [3, 4]]:
   original ──> [ * , * , * ]           copy ──> [ * , * , * ]
                 │   │   │                         │   │   │
                 v   v   └──────────┐              v   v   │
                 1   2              v              1   2   │
                                 [3, 4] <──────────────────┘
   The inner list [3,4] is SHARED. Mutating it shows in both.
   Use copy.deepcopy() to fully duplicate nested structures.
```

```python
import copy
deep = copy.deepcopy(nested)   # fully independent recursive copy
```

### Nested lists (matrices)

```python
matrix = [[1, 2, 3],
          [4, 5, 6]]
matrix[1][2]        # -> 6   (row 1, col 2)

# WRONG way to build a grid (all rows are the same object):
grid = [[0] * 3] * 2      # mutating grid[0][0] changes grid[1][0] too!
# RIGHT way:
grid = [[0] * 3 for _ in range(2)]
```

## 2. Tuples

Ordered, immutable sequences. Use for fixed collections and as dict/set keys.

```python
point = (3, 4)
single = (5,)          # trailing comma REQUIRED for 1-tuple; (5) is just int 5
empty = ()
packed = 1, 2, 3       # parentheses optional -> (1, 2, 3)
```

### Unpacking (extremely common in data code)

```python
x, y = point                 # x=3, y=4
a, b, c = (1, 2, 3)
first, *middle, last = (1, 2, 3, 4, 5)   # first=1, middle=[2,3,4], last=5
for name, age in [("Ada", 36), ("Alan", 41)]:
    print(name, age)

# Functions returning multiple values actually return a tuple:
def minmax(xs):
    return min(xs), max(xs)
lo, hi = minmax([3, 1, 4])
```

Immutability is shallow: a tuple containing a list can still have that list
mutated.

```python
t = (1, [2, 3])
t[1].append(4)      # allowed -> (1, [2, 3, 4]); the tuple itself didn't change
# t[0] = 9          # TypeError: tuples are immutable
```

### Named tuples (readable records)

```python
from collections import namedtuple
Point = namedtuple("Point", ["x", "y"])
p = Point(3, 4)
p.x, p.y            # 3, 4  (attribute access, still a tuple)
p[0]                # 3     (index access still works)
```

## 3. Dictionaries

Key -> value mappings. Hash-based, so lookups are O(1) average. Keys must be
hashable (immutable): strings, numbers, tuples work; lists do not.

```python
person = {"name": "Ada", "age": 36}
empty = {}
from_pairs = dict([("a", 1), ("b", 2)])
from_kwargs = dict(name="Ada", age=36)
```

### Access

```python
person["name"]              # -> 'Ada'  (KeyError if key missing)
person.get("email")         # -> None   (safe: no error)
person.get("email", "n/a")  # -> 'n/a'  (default if missing)
person["email"] = "a@b.c"   # add or update
"name" in person            # -> True   (membership tests KEYS)
```

`.get` vs `[]`: use `[]` when a missing key is a bug (fail loudly); use `.get`
when absence is expected.

### Iteration

```python
for key in person:               # iterates KEYS by default
    print(key)
for key in person.keys():        # explicit keys
    ...
for value in person.values():    # values
    ...
for key, value in person.items():  # both (most common)
    print(key, value)
```

### Useful methods

```python
person.update({"age": 37, "city": "London"})   # merge/overwrite
person.pop("city")            # remove & return value
person.setdefault("hits", 0)  # get, or set to default if absent
list(person.keys())           # snapshot of keys
{**d1, **d2}                  # merge dicts (d2 wins on conflict)
d1 | d2                       # merge operator (Python 3.9+)
```

### Counting and grouping patterns

```python
# Count occurrences:
from collections import Counter
Counter("abracadabra")    # Counter({'a':5,'b':2,'r':2,'c':1,'d':1})
Counter(["x","y","x"]).most_common(1)   # [('x', 2)]

# Group with defaultdict (no key-existence checks needed):
from collections import defaultdict
groups = defaultdict(list)
for word in ["apple", "banana", "avocado"]:
    groups[word[0]].append(word)
# groups -> {'a': ['apple','avocado'], 'b': ['banana']}
```

`defaultdict(list)` auto-creates an empty list for a missing key on first access,
eliminating boilerplate `if key not in d` checks.

## 4. Sets

Unordered collections of unique, hashable elements. O(1) membership.

```python
s = {1, 2, 3, 3}       # -> {1, 2, 3}  (duplicate dropped)
empty = set()          # NOT {} -- that is an empty dict!
from_list = set([1, 1, 2])   # dedup -> {1, 2}
```

### Operations

```python
s.add(4)
s.discard(10)     # remove if present, no error if absent
s.remove(2)       # remove, KeyError if absent
5 in s            # membership -> bool (very fast)
```

### Set algebra

```python
a = {1, 2, 3}
b = {2, 3, 4}
a | b        # union         -> {1, 2, 3, 4}
a & b        # intersection  -> {2, 3}
a - b        # difference    -> {1}
a ^ b        # symmetric diff-> {1, 4}
a <= b       # subset test
```

```
        a = {1,2,3}        b = {2,3,4}
        ┌───────────┐    ┌───────────┐
        │  1        │ 2  │        4  │
        │        3  │    │           │
        └───────────┴────┴───────────┘
   a & b = {2,3}   (overlap)
   a | b = {1,2,3,4} (everything)
   a - b = {1}     (in a, not b)
   a ^ b = {1,4}   (in exactly one)
```

Common use: deduplicate and fast membership.

```python
unique = list(set(items))          # dedup (order not preserved)
# preserve order while deduping:
seen = set()
result = [x for x in items if not (x in seen or seen.add(x))]
```

`frozenset` is the immutable, hashable version (can be a dict key or set element).

## 5. Comprehensions

Concise construction of lists/dicts/sets from iterables. Faster and clearer than
equivalent loops for simple transformations.

### List comprehension

```python
# Pattern: [ expression for item in iterable if condition ]
squares = [x**2 for x in range(10)]
evens = [x for x in range(20) if x % 2 == 0]
labels = [f"item {i}" for i in range(3)]

# Equivalent explicit loop (for comparison):
squares = []
for x in range(10):
    squares.append(x**2)
```

### With condition and transform

```python
# if filters; if/else transforms (note position differs):
[x for x in range(10) if x % 2 == 0]          # filter: keep evens
[x if x % 2 == 0 else -x for x in range(10)]  # transform: negate odds
```

```
   FILTER  (if at the end):     [ expr    for x in it   if cond ]
   TERNARY (if/else up front):  [ a if c else b   for x in it ]
```

### Nested comprehensions

```python
matrix = [[1, 2, 3], [4, 5, 6]]
flat = [x for row in matrix for x in row]       # -> [1,2,3,4,5,6]
# read left-to-right as nested loops:
#   for row in matrix:
#       for x in row:

transposed = [[row[i] for row in matrix] for i in range(3)]
# -> [[1,4],[2,5],[3,6]]
```

### Dict and set comprehensions

```python
{k: v for k, v in [("a",1),("b",2)]}       # dict comprehension
{x: x**2 for x in range(5)}                # {0:0,1:1,2:4,3:9,4:16}
{word: len(word) for word in ["a","bb"]}   # {'a':1,'bb':2}
{x % 3 for x in range(10)}                 # set comprehension -> {0,1,2}

# Invert a dict:
inverted = {v: k for k, v in original.items()}
```

### Generator expressions (lazy, memory-efficient)

Use parentheses instead of brackets. Values are produced on demand, not stored.

```python
gen = (x**2 for x in range(1_000_000))   # no memory spike; nothing computed yet
sum(x**2 for x in range(1000))           # parentheses optional as sole argument
total = sum(len(line) for line in open("big.txt"))  # stream a file
```

Trade-off: comprehensions build the whole collection in memory (fast repeated
access); generators use near-constant memory but can be consumed only once. For
large or streamed data, prefer generators. Covered further in file 05.

## 6. Choosing the right structure (performance)

```
   Operation           list        dict/set
   ─────────────────────────────────────────
   x in collection     O(n)        O(1)
   append/add          O(1)*       O(1)
   index access [i]    O(1)        n/a (dict: O(1) by key)
   insert at front     O(n)        n/a
   * amortized
```

If you repeatedly test membership on a large collection, convert it to a set
first. This turns an O(n*m) loop into O(n).

```python
# SLOW: O(n*m)
common = [x for x in big_list if x in another_big_list]
# FAST: O(n+m)
lookup = set(another_big_list)
common = [x for x in big_list if x in lookup]
```

For fixed-size queues, use `collections.deque` (O(1) at both ends):

```python
from collections import deque
q = deque([1, 2, 3])
q.appendleft(0)    # O(1) (list.insert(0, x) is O(n))
q.popleft()        # O(1)
```

## Exercises

1. Given a list of words, build a dict mapping each word length to the list of
   words of that length (use `defaultdict`).
2. From two lists `keys` and `values`, build a dict (hint: `zip`).
3. Deduplicate a list while preserving order.
4. Given `matrix = [[1,2],[3,4],[5,6]]`, transpose it with a comprehension.
5. Count word frequency in a paragraph and print the top 3 (hint: `Counter`).

Proceed to `03_python_functions_modules.md`.
