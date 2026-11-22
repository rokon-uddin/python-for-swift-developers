# 03. Functions and Advanced Language Features

[← 02. Swift vs Python](02-swift-vs-python.md) · [Contents](../README.md) · [04. Object-Oriented Programming →](04-oop.md)

> **In Swift terms:** functions are first-class values just like Swift closures. What changes is how arguments are named (keywords instead of labels), how defaults are evaluated, and the tools Python builds on top of functions: decorators, generators and context managers.

## Parameters: positional, keyword, defaults, `*args`, `**kwargs`

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
func fetch(_ url: String,
           timeout: Double = 10,
           retries: Int = 3) -> String {
    "\(url) \(timeout) \(retries)"
}
fetch("https://x.dev", retries: 5)    // label required

func log(_ items: String...) {          // variadic
    print(items.joined(separator: " "))
}
log("a", "b")
```

</td>
<td>

```python
def fetch(url: str, /, *, timeout: float = 10, retries: int = 3) -> str:
    return f"{url} {timeout} {retries}"

fetch("https://x.dev", retries=5)      # keyword required after `*`

def log(*items: str) -> None:           # variadic → tuple
    print(" ".join(items))

log("a", "b")
```

</td>
</tr>
</table>

Swift's argument labels ≈ Python keyword arguments. Without a bare `*`, callers may pass any argument either positionally or by name, so add `*` when you want Swift-style mandatory labels. `/` marks parameters that can *only* be positional (like Swift's `_`).

```python
def make_order(item, size="M", /, *extras, milk=True, **meta):
    #          ^ positional-only ^  ^ varargs ^ keyword-only ^ extra keywords
    print(item, size, extras, milk, meta)

make_order("latte")                                  # latte M () True {}
make_order("latte", "L", "vanilla", milk=False, table=4)
# latte L ('vanilla',) False {'table': 4}
```

| Marker | Meaning |
|---|---|
| `/` | Everything before it is positional-only (3.8+). Lets you rename parameters without breaking callers |
| `*args` | Extra positional arguments, collected into a `tuple` |
| `*` (bare) | Everything after it is keyword-only |
| `**kwargs` | Extra keyword arguments, collected into a `dict` |

Unpacking works at the call site too:

```python
args = ("latte", "L")
kwargs = {"milk": False}
make_order(*args, **kwargs)
```

## Mutable defaults: evaluated once

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
func addTag(_ tag: String,
            to tags: [String] = []) -> [String] {
    var tags = tags          // default is rebuilt on every call
    tags.append(tag)
    return tags
}
addTag("a")                  // ["a"]
addTag("b")                  // ["b"]
```

</td>
<td>

```python
def add_tag(tag, tags=[]):   # ONE list, created when `def` runs
    tags.append(tag)
    return tags

add_tag("a")                 # ['a']
print(add_tag("b"))          # → ['a', 'b']  same list again!
```

</td>
</tr>
</table>

Swift evaluates a default argument on every call. Python evaluates it once, when the function is defined, and reuses that object. Use `None` as the default for lists, dicts and sets.

The fix:

```python
def add_tag(tag: str, tags: list[str] | None = None) -> list[str]:
    if tags is None:
        tags = []          # a fresh list on every call, like Swift
    tags.append(tag)
    return tags
```

## Return values

- No `return` (or bare `return`) → returns `None`.
- Multiple values → a tuple.
- Code after `return` is unreachable. Early returns (guard-clause style) are idiomatic.

## Scope: LEGB, `global`, `nonlocal`

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
var status = "global"

func outer() -> String {
    var status = "enclosing"     // shadows the global
    func inner() {
        status = "changed"       // just works: closure captures it
    }
    inner()
    return status
}
```

</td>
<td>

```python
status = "global"

def outer() -> str:
    status = "enclosing"
    def inner() -> None:
        nonlocal status          # without this, `status = ...` would
        status = "changed"       # create a NEW local variable
    inner()
    return status

print(outer(), status)           # → changed global
```

</td>
</tr>
</table>

In Python, **assigning** to a name inside a function makes it local to that function unless you declare `nonlocal` (enclosing function) or `global` (module). Reading outer variables needs no keyword.

Name lookup order: **L**ocal → **E**nclosing function → **G**lobal (module) → **B**uilt-in.

⚠️ Assigning to a name anywhere inside a function makes it local for the **whole** function:

```python
count = 0
def bump():
    count += 1      # UnboundLocalError: count is local, read before assignment
```

Mutating module-level state with `global` makes code hard to test. I treat `global` as a code smell outside of small scripts; pass state explicitly or wrap it in a class.

## Pure vs impure functions, recursion

Pure functions (output depends only on input, no side effects) are easier to test and cache with `functools.cache`. Python has **no tail-call optimisation**; default recursion limit is ~1000 frames (`sys.getrecursionlimit()`). Prefer iteration for deep recursion.

## Lambdas and higher-order functions

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
struct Order { let id: Int; let total: Double }
let orders = [Order(id: 3, total: 50), Order(id: 1, total: 120)]

let byTotal = orders.sorted { $0.total > $1.total }
let ids = orders.map(\.id)
```

</td>
<td>

```python
from operator import attrgetter
from dataclasses import dataclass

@dataclass
class Order:
    id: int
    total: float

orders = [Order(3, 50), Order(1, 120)]

by_total = sorted(orders, key=lambda o: o.total, reverse=True)
ids = [o.id for o in orders]
by_id = sorted(orders, key=attrgetter("id"))   # like \.id
```

</td>
</tr>
</table>

Python sorts with a **key function** ("what to compare") instead of a comparator closure ("is a before b?"). `reverse=True` flips the order.

Functions are first-class objects: pass them, return them, store them in dicts. They carry metadata (`__name__`, `__doc__`, `__annotations__`, `__defaults__`).

## Closures

A closure is a nested function that references variables from an enclosing scope — the same idea as a Swift closure. The side-by-side version, including `nonlocal` for mutating captured state, is in [Swift vs Python → Closures](02-swift-vs-python.md#closures-and-lambdas).

```python
def multiplier(factor: float):
    def apply(x: float) -> float:
        return x * factor
    return apply

double = multiplier(2)
print(double(21))   # → 42
```

## Decorators

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
import Foundation

// Swift has no direct equivalent; the closest is a
// higher-order function you call explicitly:
func timed<T>(_ label: String,
              _ body: () throws -> T) rethrows -> T {
    let start = Date()
    defer { print(label, Date().timeIntervalSince(start)) }
    return try body()
}

let result = timed("render") { render("home") }
```

</td>
<td>

```python
import functools, time

def timed(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        try:
            return func(*args, **kwargs)
        finally:
            print(func.__name__, time.perf_counter() - start)
    return wrapper

@timed                     # applied ONCE, at definition time
def render(page: str) -> str:
    return f"<{page}>"

result = render("home")    # every call is now timed
```

</td>
</tr>
</table>

A decorator is that wrapper function applied permanently at the definition site: `@timed` above `def render` means `render = timed(render)`. Think of it as a property wrapper for functions.

Always use `functools.wraps`; without it the wrapped function loses its `__name__`, docstring and signature, which breaks logging, debugging and frameworks that introspect functions. The fully typed version below keeps the wrapped function's signature visible to the type checker (`ParamSpec` ≈ Swift's parameter packs for this purpose):

```python
import functools
import logging
import time
from collections.abc import Callable
from typing import ParamSpec, TypeVar

P = ParamSpec("P")
R = TypeVar("R")

def timed(func: Callable[P, R]) -> Callable[P, R]:
    @functools.wraps(func)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        start = time.perf_counter()
        try:
            return func(*args, **kwargs)
        finally:
            logging.info("%s took %.3fs", func.__name__, time.perf_counter() - start)
    return wrapper

@timed
def render(template: str, *, escape: bool = False) -> str:
    return f"{template} (escape={escape})"

print(render("home", escape=True), render.__name__)   # → home (escape=True) render
```

Decorator with arguments = a function that returns a decorator:

```python
import functools

def require_role(role: str):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(user: dict, *args, **kwargs):
            if user.get("role") != role:
                raise PermissionError(f"{role} required")
            return func(user, *args, **kwargs)
        return wrapper
    return decorator

@require_role("admin")
def read_inventory(user: dict) -> list[str]:
    return ["sku-1", "sku-2"]

print(read_inventory({"role": "admin"}))
```

Built-in decorators worth knowing: `@property`, `@staticmethod`, `@classmethod`, `@functools.cache`, `@functools.lru_cache(maxsize=...)`, `@functools.cached_property`, `@functools.singledispatch`, `@dataclass`, `@contextlib.contextmanager`, `@typing.override` (3.12+).

## Generators

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
struct Batches: Sequence, IteratorProtocol {
    let rows: [Int]
    let size: Int
    var index = 0

    mutating func next() -> [Int]? {
        guard index < rows.count else { return nil }
        defer { index += size }
        return Array(rows[index..<Swift.min(index + size, rows.count)])
    }
}

for batch in Batches(rows: Array(0..<7), size: 3) {
    print(batch)
}
```

</td>
<td>

```python
def batches(rows: list[int], size: int):
    for i in range(0, len(rows), size):
        yield rows[i : i + size]       # pause here, resume on next()

for batch in batches(list(range(7)), 3):
    print(batch)
# [0, 1, 2]
# [3, 4, 5]
# [6]
```

</td>
</tr>
</table>

`yield` replaces the whole `Sequence`/`IteratorProtocol` boilerplate. Python keeps the function's local variables alive between values, so you write a normal loop.

A function containing `yield` returns a **generator** — a lazy, resumable iterator. Nothing runs until the first `next()`.

Infinite generators are fine because values are produced on demand:

```python
import itertools

def ids(prefix: str):
    n = 1
    while True:
        yield f"{prefix}-{n}"
        n += 1

print(list(itertools.islice(ids("order"), 3)))   # → ['order-1', 'order-2', 'order-3']
```

Each call creates an independent generator with its own state.

**Generator protocol:** `send()`, `throw()`, `close()`.

```python
def running_average():
    total, count = 0.0, 0
    average = None
    while True:
        value = yield average      # receives the value passed to send()
        total += value
        count += 1
        average = total / count

avg = running_average()
next(avg)                    # "prime" the generator: run to the first yield
avg.send(10); print(avg.send(20))   # → 15.0
avg.close()                  # raises GeneratorExit inside, runs cleanup
```

⚠️ Catch `GeneratorExit` explicitly if you need cleanup — or better, use `try/finally`. A bare `except:` inside a generator swallows `GeneratorExit` and can turn `close()` into `RuntimeError: generator ignored GeneratorExit`.

**`yield from`** delegates to a sub-iterator (and forwards `send`/`throw`):

```python
def core_services():   yield from ("cache", "db")
def edge_services():   yield from ("cdn", "gateway")
def all_services():
    yield from core_services()
    yield from edge_services()

print(list(all_services()))   # → ['cache', 'db', 'cdn', 'gateway']
```

**Swift:** the closest analogue is a custom `Sequence`/`IteratorProtocol`, or `AsyncStream` for the async version. Python generators are far less ceremony.

## Context managers

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
import Foundation

func readConfig(at path: String) throws -> String {
    let handle = try FileHandle(forReadingFrom: URL(fileURLWithPath: path))
    defer { try? handle.close() }        // cleanup on every exit path
    let data = handle.readDataToEndOfFile()
    return String(decoding: data, as: UTF8.self)
}
```

</td>
<td>

```python
def read_config(path: str) -> str:
    with open(path, encoding="utf-8") as handle:   # closed on every exit path
        return handle.read()
```

</td>
</tr>
</table>

`with` packages a `defer` together with the resource. Objects that support it (files, locks, DB sessions, HTTP clients) clean themselves up when the block ends, even if an exception is raised.

`with` guarantees setup/teardown — Python's `defer` + RAII. Anything with `__enter__`/`__exit__` works.

```python
from contextlib import contextmanager
import time

@contextmanager
def timer(label: str):
    start = time.perf_counter()
    try:
        yield                                  # body of the with-block runs here
    finally:
        print(f"{label}: {time.perf_counter() - start:.3f}s")

with timer("load"), open(__file__) as f:       # multiple managers on one line
    f.read()
```

Class form, when you need state or to suppress exceptions:

```python
class Transaction:
    def __enter__(self):
        print("BEGIN")
        return self
    def __exit__(self, exc_type, exc, tb) -> bool:
        print("ROLLBACK" if exc_type else "COMMIT")
        return False          # True would swallow the exception

with Transaction():
    pass
```

Useful helpers in `contextlib`: `suppress`, `closing`, `ExitStack` (dynamic number of managers), `redirect_stdout`, `chdir` (3.11+), `nullcontext`, `asynccontextmanager`.

## Dunder (special) methods

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
import Foundation

struct Money: Equatable, Hashable, CustomStringConvertible {
    let cents: Int

    static func + (a: Money, b: Money) -> Money {
        Money(cents: a.cents + b.cents)
    }
    var description: String {
        String(format: "$%.2f", Double(cents) / 100)
    }
}
print(Money(cents: 150) + Money(cents: 275))   // $4.25
```

</td>
<td>

```python
from dataclasses import dataclass

@dataclass(frozen=True)            # generates __eq__ and __hash__
class Money:
    cents: int

    def __add__(self, other: "Money") -> "Money":    # static func +
        return Money(self.cents + other.cents)

    def __str__(self) -> str:                        # description
        return f"${self.cents / 100:.2f}"

print(Money(150) + Money(275))     # → $4.25
```

</td>
</tr>
</table>

Swift uses protocols (`Equatable`, `CustomStringConvertible`) and `static func` operators. Python uses specially named methods. The table below maps them.

| Swift | Python dunder |
|---|---|
| `Equatable` `==` | `__eq__` |
| `Hashable` | `__hash__` |
| `Comparable` `<` | `__lt__` (plus `functools.total_ordering`) |
| `CustomStringConvertible.description` | `__str__` |
| `CustomDebugStringConvertible` | `__repr__` |
| `Collection.count` | `__len__` |
| `subscript` | `__getitem__` / `__setitem__` |
| `Sequence.makeIterator()` | `__iter__` |
| `contains(_:)` | `__contains__` (used by `in`) |
| `callAsFunction()` | `__call__` |
| `static func +` | `__add__`; `__radd__` (reflected), `__iadd__` (`+=`) |
| `Bool` conversion (no Swift equivalent) | `__bool__` (falls back to `__len__`) |
| `@dynamicMemberLookup` | `__getattr__` (called only when normal lookup fails) |
| `deinit` / scope exit | `__enter__` / `__exit__` (context managers) |

`+` on lists/strings is just `__add__` — "operator overloading" is the normal way built-ins work.

---

[← 02. Swift vs Python](02-swift-vs-python.md) · [Contents](../README.md) · [04. Object-Oriented Programming →](04-oop.md)
