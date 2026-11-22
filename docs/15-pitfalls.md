# 15. Common Pitfalls and Best Practices

[← 14. Python for LLM and AI Application Development](14-llm-applications.md) · [Contents](../README.md) · [16. Practical Code Recipes →](16-recipes.md)

> **In Swift terms:** most of these bugs exist because Python lacks something the Swift compiler gives you for free: value semantics, per-iteration loop variables, compile-time type and exhaustiveness checks, and a ban on blocking inside `async`.

| Pitfall | What Swift does | What Python does |
|---|---|---|
| [Mutable default arguments](#mutable-default-arguments) | Re-evaluates defaults on each call | Evaluates once and reuses the object |
| [Shallow vs deep copying](#shallow-vs-deep-copying) | Nested arrays/structs copy as values | Copies are shallow; nested lists are shared |
| [Late binding in closures](#late-binding-in-closures) | Each `for` iteration has its own constant | One loop variable, read when the closure runs |
| [Variable scope](#variable-scope) | Assignment to an outer `var` just works | Assignment creates a local unless `nonlocal`/`global` |
| [Floating-point precision](#floating-point-precision) | Same IEEE doubles; `Decimal` in Foundation | Same; plus `round()` uses banker's rounding |
| [Iterator exhaustion](#iterator-exhaustion) | Arrays re-iterate; iterators are explicit | Generators silently produce nothing the second time |
| [Modifying while iterating](#modifying-a-collection-while-iterating) | You iterate a copy (value semantics) | `RuntimeError` for dicts, skipped items for lists |
| [Blocking in asyncio](#blocking-operations-in-asyncio) | Blocking hurts one pool thread | Blocking freezes every coroutine |
| [Exception anti-patterns](#exception-handling-anti-patterns) | Compiler forces `try` at throwing calls | Errors are invisible in signatures and easy to swallow |

## Mutable default arguments

Covered in [section 03](03-functions.md#mutable-defaults-evaluated-once): defaults are evaluated once. Use `None` and create the object inside the function, or `field(default_factory=list)` in dataclasses.

## Shallow vs deep copying

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
var config = ["db": ["host": "localhost"]]
var copy = config              // value semantics, all the way down
copy["db"]!["host"] = "prod"
print(config["db"]!["host"]!)  // localhost
```

</td>
<td>

```python
import copy

config = {"db": {"host": "localhost"}}
shallow = config.copy()             # new outer dict, SAME inner dict
shallow["db"]["host"] = "prod"
print(config["db"]["host"])         # → prod   (changed!)

config = {"db": {"host": "localhost"}}
deep = copy.deepcopy(config)        # ≈ Swift's copy
deep["db"]["host"] = "prod"
print(config["db"]["host"])         # → localhost
```

</td>
</tr>
</table>

## Late binding in closures

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
var callbacks: [() -> Int] = []
for i in 0..<3 {
    callbacks.append { i }   // each iteration has its own `i`
}
print(callbacks.map { $0() })   // [0, 1, 2]
```

</td>
<td>

```python
callbacks = []
for i in range(3):
    callbacks.append(lambda: i)    # all three share ONE `i`
print([cb() for cb in callbacks])  # → [2, 2, 2]

callbacks = [lambda i=i: i for i in range(3)]   # freeze the value
print([cb() for cb in callbacks])  # → [0, 1, 2]
```

</td>
</tr>
</table>

Python closures read a captured variable when they **run**, not when they're created. Swift gives each `for` iteration its own `i`; Python's loop reuses one variable, so every closure sees its final value. Freeze the value with a default argument or `functools.partial(fn, i)`.

## Variable scope

- Assigning inside a function makes the name local everywhere in that function (`UnboundLocalError`).
- Loop variables leak: after `for i in range(3): pass`, `i == 2`. Comprehension variables don't leak.
- Shadowing built-ins (`list = [...]`, `type = "x"`, `id = 5`, `input = ...`) breaks later code in the same scope. Linters flag it (`A` rules in Ruff).

## Circular imports

`a.py` imports `b.py` which imports `a.py` → one of them sees a half-initialised module (`ImportError: cannot import name ... (most likely due to a circular import)`).

Fixes, in order of preference:

1. Move the shared code into a third module both can import.
2. Depend on abstractions (`Protocol`) defined in a lower-level module.
3. Import inside the function that needs it.
4. For type-only imports: `from typing import TYPE_CHECKING` / `if TYPE_CHECKING: from .b import B`.

## Floating-point precision

```python
from decimal import Decimal, ROUND_HALF_UP

print(round(2.675, 2))                        # → 2.67  (2.675 is really 2.67499999...)
print(round(0.5), round(1.5), round(2.5))     # → 0 2 2  banker's rounding (round half to even)
price = Decimal("2.675").quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)
print(price)                                  # → 2.68
```

Store money as integer minor units (cents) or `Decimal`. Construct `Decimal` from strings, never from floats.

## Iterator exhaustion

```python
rows = (r for r in range(3))
print(sum(rows), sum(rows))           # → 3 0

import itertools
a, b = itertools.tee(range(3))        # two independent iterators (buffers in memory)
values = list(range(3))               # or simply materialise if you need multiple passes
```

Also single-pass: file objects, `map`, `filter`, `zip`, `csv.reader`, database cursors, HTTP streaming bodies.

## Modifying a collection while iterating

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
var stock = ["sku-1": 0, "sku-2": 5]
for (key, qty) in stock where qty == 0 {
    stock[key] = nil           // fine: the loop iterates a copy
}
```

</td>
<td>

```python
stock = {"sku-1": 0, "sku-2": 5}
# for key in stock: del stock[key]  → RuntimeError
for key in list(stock):            # iterate over a snapshot
    if stock[key] == 0:
        del stock[key]
print(stock)                       # → {'sku-2': 5}
```

</td>
</tr>
</table>

```python
stock = {"sku-1": 0, "sku-2": 5, "sku-3": 0}
# for k in stock: if stock[k] == 0: del stock[k]   → RuntimeError
stock = {k: v for k, v in stock.items() if v}     # build a new one
items = [1, 2, 3, 4]
items[:] = [x for x in items if x % 2]            # in-place filter of a list
```

## Memory management: reference counting and GC

- Objects are freed as soon as their reference count hits zero (deterministic, like ARC) — which is why files are often closed "accidentally" in CPython. Don't rely on it; use `with`.
- The **cyclic GC** (`gc` module) periodically collects reference cycles. You can `gc.freeze()` after startup in forking servers or disable it temporarily during bulk allocation, but measure first.
- `sys.getrefcount(obj)`, `sys.getsizeof(obj)` (shallow), and `tracemalloc` for finding leaks.
- Real "leaks" in Python are almost always unbounded caches, global lists that keep growing, or `@lru_cache` on methods.
- `__del__` finalisers run at unpredictable times; prefer context managers or `weakref.finalize`.

## GIL limitations

Threads won't speed up CPU-bound Python. Use processes, vectorised libraries, or a free-threaded build — see [section 10](10-concurrency.md#the-global-interpreter-lock-gil). Conversely, don't reach for processes for I/O-bound work; threads or asyncio are much cheaper.

## Blocking operations in asyncio

The most common asyncio bug: a synchronous call inside a coroutine stalls the whole event loop.

| Blocking (don't) | Non-blocking (do) |
|---|---|
| `time.sleep(1)` | `await asyncio.sleep(1)` |
| `requests.get(url)` | `await httpx.AsyncClient().get(url)` / `aiohttp` |
| Sync DB driver (`psycopg2`, `pymongo`) | Async driver (`asyncpg`, `psycopg` async, `motor`/async PyMongo) or `asyncio.to_thread` |
| Heavy CPU work | `loop.run_in_executor(process_pool, fn, ...)` |
| `open(...).read()` of a big file | `asyncio.to_thread(path.read_bytes)` or `aiofiles` |
| Calling `asyncio.run()` inside a running loop | `await` the coroutine; one `asyncio.run` per program |

Detect it: `asyncio.run(main(), debug=True)` (or `PYTHONASYNCIODEBUG=1`) logs callbacks that take longer than 100 ms. Ruff's `ASYNC` rules catch common cases statically.

Calling an `async def` without `await` just creates a coroutine object and does nothing (you'll see `RuntimeWarning: coroutine ... was never awaited`).

## Dependency conflicts

- One virtual environment per project. Never install into the system interpreter.
- Use a lockfile (`uv.lock`, `poetry.lock`, compiled `requirements.txt`) for reproducible installs; declare ranges in `pyproject.toml`.
- `pip check` reports broken requirements; `uv tree` / `pipdeptree` show who pulled in what.
- Big AI stacks (torch, transformers, langchain*, provider SDKs) move fast and pin aggressively — upgrade them together and run tests.
- Pin the Python version (`requires-python`, `.python-version`) — wheels for heavy packages often lag new releases.

## Exception-handling anti-patterns

```python
# ✗ Bare except: also catches KeyboardInterrupt, SystemExit, GeneratorExit
try: ...
except: pass

# ✗ Catch-all that hides bugs
try:
    total = compute()
except Exception:
    total = 0

# ✗ Losing the original traceback
try: ...
except KeyError as e:
    raise ValueError("bad config")           # use `from e`

# ✓ Narrow, logged, re-raised with context
import logging
log = logging.getLogger(__name__)

def load(path: str) -> str:
    try:
        with open(path, encoding="utf-8") as f:
            return f.read()
    except FileNotFoundError:
        log.warning("config %s missing, using defaults", path)
        return "{}"
    except PermissionError as e:
        raise RuntimeError(f"cannot read {path}") from e
```

Catching `Exception` is fine at a **process boundary** (a request handler, a worker loop, a CLI `main`) — as long as you log with `logger.exception` and fail the unit of work, not silently continue.

## More rules I follow

- `is None`, never `== None`; never `is` for numbers or strings.
- `isinstance(x, (int, float))` rather than `type(x) == int` (respects subclasses).
- `with` for every resource: files, locks, connections, sessions, pools.
- `if __name__ == "__main__":` in every script, mandatory with multiprocessing.
- `encoding="utf-8"` on every text `open()` (until 3.15 is your floor).
- `timeout=` on every network call.
- Parameterised SQL; `subprocess.run([...])` without `shell=True`.
- `secrets` for anything security-sensitive; `random` for simulations only.
- `datetime.now(timezone.utc)`; store UTC, convert at the edges.
- Don't name your own files after stdlib or popular modules (`random.py`, `json.py`, `test.py`, `openai.py`) — they shadow the real import.

---

[← 14. Python for LLM and AI Application Development](14-llm-applications.md) · [Contents](../README.md) · [16. Practical Code Recipes →](16-recipes.md)
