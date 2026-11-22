# 08. Functional Programming and Pythonic Patterns

[← 07. Modules, Packages and Dependency Management](07-modules-and-packaging.md) · [Contents](../README.md) · [09. File Handling and Data Serialization →](09-files-and-serialization.md)

> **In Swift terms:** everything you do with `map`, `filter`, `reduce`, `compactMap`, `sorted(by:)` and `Dictionary(grouping:)` has a Python counterpart, but idiomatic Python usually prefers comprehensions and built-in functions (`sum`, `any`, `all`, `sorted`) over method chains.

## `map`, `filter`, `reduce` — and why comprehensions usually win

| Swift | Idiomatic Python |
|---|---|
| `xs.map { $0 * 2 }` | `[x * 2 for x in xs]` |
| `xs.filter { $0 > 0 }` | `[x for x in xs if x > 0]` |
| `xs.compactMap { Int($0) }` | `[int(s) for s in xs if s.isdigit()]` |
| `xs.flatMap { $0 }` | `[y for x in xs for y in x]` |
| `xs.reduce(0, +)` | `sum(xs)` |
| `xs.reduce(1, *)` | `math.prod(xs)` |
| `xs.contains(where: { $0 > 5 })` | `any(x > 5 for x in xs)` |
| `xs.allSatisfy { $0 > 0 }` | `all(x > 0 for x in xs)` |
| `xs.first(where: { $0 > 5 })` | `next((x for x in xs if x > 5), None)` |
| `xs.count(where: { $0 > 5 })` | `sum(1 for x in xs if x > 5)` |
| `xs.max()`, `xs.min()` | `max(xs)`, `min(xs)` (raise on empty; pass `default=`) |
| `xs.sorted(by: { $0.age < $1.age })` | `sorted(xs, key=lambda p: p.age)` |
| `xs.lazy.map { … }` | generator expression `(… for x in xs)` |
| `Dictionary(grouping: xs, by: \.dept)` | `defaultdict(list)` loop or `itertools.groupby` (sorted input) |
| `Array(xs.prefix(3))` | `xs[:3]` |
| `xs.chunks(ofCount: 3)` (Algorithms) | `itertools.batched(xs, 3)` |

```python
from functools import reduce
import operator

prices = [120, 80, 45, 300]
with_tax = list(map(lambda p: p * 1.1, prices))
cheap = list(filter(lambda p: p < 100, prices))
product = reduce(operator.mul, [1, 2, 3, 4])         # → 24

# Idiomatic equivalents
with_tax = [p * 1.1 for p in prices]
cheap = [p for p in prices if p < 100]
```

`map`/`filter` return lazy iterators. `map(str, items)` is perfectly readable when you already have a named function; with a `lambda`, the comprehension reads better.

## `zip`, `enumerate`, `any`, `all`

```python
names = ["Ada", "Linus", "Grace"]
bills = [50, 70, 100]

for i, name in enumerate(names, start=1):
    print(i, name)

for name, amount in zip(names, bills, strict=True):   # strict: raise on length mismatch (3.10+)
    print(f"{name} paid {amount}")

lookup = dict(zip(names, bills))
cols = list(zip(*[(1, "a"), (2, "b")]))   # transpose → [(1, 2), ('a', 'b')]

any(b > 90 for b in bills)    # → True  (short-circuits)
all(b > 10 for b in bills)    # → True
```

⚠️ Without `strict=True`, `zip` silently truncates to the shortest input.

## Sorting

```python
from dataclasses import dataclass
from operator import attrgetter

@dataclass
class Employee:
    name: str
    dept: str
    salary: int

staff = [Employee("Ana", "eng", 120), Employee("Bo", "ops", 90), Employee("Cy", "eng", 150)]

sorted(staff, key=attrgetter("dept", "salary"))                # multi-key
sorted(staff, key=lambda e: (e.dept, -e.salary))               # dept asc, salary desc
max(staff, key=attrgetter("salary")).name                      # → 'Cy'
```

Python's sort (Timsort/Powersort) is **stable**, so you can sort by secondary key first, then primary.

## `collections`

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
let words = ["a", "b", "a"]
let counts = words.reduce(into: [:]) { $0[$1, default: 0] += 1 }

let byDept = Dictionary(grouping: staff, by: \.dept)
```

</td>
<td>

```python
from collections import Counter, defaultdict

words = ["a", "b", "a"]
counts = Counter(words)                  # Counter({'a': 2, 'b': 1})

by_dept: defaultdict[str, list[str]] = defaultdict(list)
for name, dept in [("Ana", "eng"), ("Bo", "ops")]:
    by_dept[dept].append(name)
```

</td>
</tr>
</table>

```python
from collections import Counter, deque, namedtuple, ChainMap

Counter("mississippi").most_common(2)            # → [('i', 4), ('s', 4)]

recent = deque(maxlen=3)                         # O(1) appends/pops at both ends; bounded buffer
for x in range(5):
    recent.append(x)
print(recent)                                    # → deque([2, 3, 4], maxlen=3)

Point = namedtuple("Point", ["x", "y"])          # lightweight immutable record (or typing.NamedTuple)
settings = ChainMap({"debug": True}, {"debug": False, "port": 80})  # layered lookups
print(settings["debug"], settings["port"])       # → True 80
```

`OrderedDict` is mostly historical now that `dict` is ordered, but still has `move_to_end()` — handy for LRU caches.

## `itertools`

```python
import itertools as it

list(it.chain([1, 2], [3]))                        # → [1, 2, 3]
list(it.chain.from_iterable([[1], [2, 3]]))        # flatten one level
list(it.islice(it.count(10), 3))                   # → [10, 11, 12]
list(it.batched(range(7), 3))                      # 3.12+ → [(0, 1, 2), (3, 4, 5), (6,)]
list(it.pairwise([1, 4, 9]))                       # → [(1, 4), (4, 9)]
list(it.accumulate([1, 2, 3]))                     # running sum → [1, 3, 6]
list(it.product("ab", [1, 2]))                     # cartesian product
list(it.combinations("abc", 2))                    # → [('a','b'), ('a','c'), ('b','c')]
list(it.takewhile(lambda x: x < 3, [1, 2, 5, 1]))  # → [1, 2]

rows = [("eng", "Ana"), ("eng", "Cy"), ("ops", "Bo")]
for dept, group in it.groupby(rows, key=lambda r: r[0]):   # ⚠️ input must be sorted by key
    print(dept, [name for _, name in group])
```

## `functools`

```python
from functools import cache, lru_cache, partial, singledispatch, cached_property

@cache                      # unbounded memoisation (args must be hashable)
def fib(n: int) -> int:
    return n if n < 2 else fib(n - 1) + fib(n - 2)

@lru_cache(maxsize=1024)    # bounded; .cache_info(), .cache_clear()
def load_config(path: str) -> dict: return {}

int_from_hex = partial(int, base=16)
print(int_from_hex("ff"))   # → 255

@singledispatch
def render(value) -> str: return str(value)
@render.register
def _(value: list) -> str: return ", ".join(map(render, value))
print(render([1, 2]))       # → '1, 2'
```

⚠️ `@cache` on a method holds a reference to `self` forever (memory leak for many instances). Use `cached_property` or a module-level cache keyed by an ID.

## Generator pipelines

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
let errors = lines
    .lazy
    .filter { !$0.isEmpty }
    .map { parse($0) }
    .filter { $0.level == "error" }

for e in errors.prefix(10) { print(e) }   // evaluated on demand
```

</td>
<td>

```python
def parse_lines(lines):
    for line in lines:
        if line.strip():
            yield line.split(":", 1)

lines = ["error: disk", "", "info: ok", "error: net"]
errors = (p for p in parse_lines(lines) if p[0] == "error")

for e in errors:
    print(e)          # ['error', ' disk'], then ['error', ' net']
```

</td>
</tr>
</table>

Same idea as a `lazy` sequence chain: nothing runs until the `for` loop pulls values, and only one item is in memory at a time.

Chaining generators processes a stream one item at a time with constant memory — the Python counterpart of a lazy `Sequence` chain or a Combine pipeline:

```python
from collections.abc import Iterable, Iterator
import json

def read_lines(path: str) -> Iterator[str]:
    with open(path, encoding="utf-8") as f:
        yield from f

def parse(lines: Iterable[str]) -> Iterator[dict]:
    for line in lines:
        if line.strip():
            yield json.loads(line)

def only_errors(events: Iterable[dict]) -> Iterator[dict]:
    return (e for e in events if e.get("level") == "error")

# Nothing is read until the final consumer pulls items:
# for event in only_errors(parse(read_lines("app.jsonl"))): ...
```

---

[← 07. Modules, Packages and Dependency Management](07-modules-and-packaging.md) · [Contents](../README.md) · [09. File Handling and Data Serialization →](09-files-and-serialization.md)
