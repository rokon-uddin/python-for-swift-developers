# 06. Error Handling and Debugging

[← 05. Python Type System](05-type-system.md) · [Contents](../README.md) · [07. Modules, Packages and Dependency Management →](07-modules-and-packaging.md)

> **In Swift terms:** exceptions work like `throw`/`catch`, with three differences: function signatures don't say what they throw, callers don't write `try` before each call, and forgetting to handle an error is never a compile error: an unhandled exception unwinds the stack and, if nothing catches it, ends the program with a traceback (Python's stack trace).

## `try` / `except` / `else` / `finally`

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
func parseQuantity(_ raw: String) -> Int? {
    defer { print("done") }          // always runs
    guard let qty = Int(raw) else {
        print("invalid quantity")
        return nil
    }
    print("parsed")
    return qty
}
```

</td>
<td>

```python
def parse_quantity(raw: str) -> int | None:
    try:
        qty = int(raw)                # int("x") raises ValueError
    except ValueError:
        print("invalid quantity")
        return None
    else:                             # only when no exception happened
        print("parsed")
        return qty
    finally:                          # always runs (≈ defer)
        print("done")
```

</td>
</tr>
</table>

`Int("x")` returns `nil` in Swift; `int("x")` **raises** in Python. Many Python APIs report failure by raising rather than returning an optional, so `try`/`except` shows up where Swift would use `guard let`.

Keep the `try` block small — only the line that can fail. Code that should run on success goes in `else`, so you don't accidentally catch exceptions from it.

Multiple exceptions:

```python
prices = {"latte": 20}

def cost(item: str, qty: str | int) -> int:
    try:
        return prices[item] * int(qty)
    except KeyError:
        raise LookupError(f"{item!r} is not on the menu") from None
    except (TypeError, ValueError) as exc:
        raise ValueError(f"bad quantity {qty!r}") from exc   # keep the original cause
```

`raise X from exc` sets `__cause__`, so tracebacks show both errors. `from None` hides the original when it's just noise.

## Exception hierarchy (the useful part)

```text
BaseException
 ├── SystemExit, KeyboardInterrupt, GeneratorExit   ← don't catch these by accident
 ├── BaseExceptionGroup → ExceptionGroup             (3.11+)
 └── Exception                                       ← catch from here down
      ├── ArithmeticError → ZeroDivisionError, OverflowError
      ├── LookupError → KeyError, IndexError
      ├── OSError → FileNotFoundError, PermissionError, TimeoutError, ConnectionError
      ├── TypeError, ValueError (→ UnicodeError), AttributeError, NameError
      ├── RuntimeError → RecursionError, NotImplementedError
      ├── StopIteration, StopAsyncIteration
      └── ImportError → ModuleNotFoundError
```

`asyncio.CancelledError` inherits from `BaseException` (3.8+) precisely so `except Exception` doesn't swallow cancellation.

## Custom exceptions

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
enum InventoryError: Error {
    case outOfStock(item: String, requested: Int)
}

do {
    throw InventoryError.outOfStock(item: "milk", requested: 2)
} catch let InventoryError.outOfStock(item, requested) {
    print(item, requested)
}
```

</td>
<td>

```python
class InventoryError(Exception): ...

class OutOfStockError(InventoryError):
    def __init__(self, item: str, requested: int) -> None:
        super().__init__(f"{item}: requested {requested}")
        self.item, self.requested = item, requested

try:
    raise OutOfStockError("milk", 2)
except OutOfStockError as e:       # catches subclasses too
    print(e.item, e.requested)     # → milk 2
```

</td>
</tr>
</table>

A Swift error enum with cases becomes a small class hierarchy. `except BaseClass` catches every subclass, so callers can handle one specific error or the whole family.

Define a small hierarchy per package so callers can catch at the right granularity:

Raise early and loudly on invalid state. Failing fast with a clear message beats propagating a bad value three layers deep.

## Exception groups (3.11+)

When several concurrent tasks fail, `asyncio.TaskGroup` raises an `ExceptionGroup`. Handle members by type with `except*`:

```python
try:
    raise ExceptionGroup("batch failed", [ValueError("bad row"), KeyError("id")])
except* ValueError as group:
    print("value errors:", group.exceptions)
except* KeyError as group:
    print("key errors:", group.exceptions)
```

`exc.add_note("context")` (3.11+) attaches extra info to any exception's traceback.

## Assertions

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
let index = 0, items = [1]
assert(index >= 0, "negative index")    // removed in -O builds
precondition(!items.isEmpty)            // kept in release builds
fatalError("unreachable")
```

</td>
<td>

```python
index, items = 0, [1]
assert index >= 0, "negative index"     # removed with `python -O`
if not items:                           # ≈ precondition: always runs
    raise ValueError("items must not be empty")
# raise AssertionError("unreachable")  ≈ fatalError
```

</td>
</tr>
</table>

`assert cond, "message"` is for **internal invariants**, never input validation: assertions are stripped when Python runs with `-O`.

## Logging

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
import os

let logger = Logger(subsystem: "app", category: "orders")
logger.info("processing \(count) rows")
logger.error("calculation failed: \(error)")
```

</td>
<td>

```python
import logging

logger = logging.getLogger("orders")     # usually getLogger(__name__)
logger.info("processing %d rows", 120)
logger.error("calculation failed: %s", "division by zero")
```

</td>
</tr>
</table>

Use `logging`, not `print`, for anything that outlives a script.

```python
import logging

logger = logging.getLogger(__name__)          # one logger per module

def configure_logging(level: int = logging.INFO) -> None:
    logging.basicConfig(
        level=level,
        format="%(asctime)s %(levelname)-8s %(name)s: %(message)s",
    )

configure_logging()
logger.info("processing %d rows", 120)        # lazy %-formatting: no work if level is disabled
try:
    1 / 0
except ZeroDivisionError:
    logger.exception("calculation failed")    # logs at ERROR with the traceback
```

- Configure logging **once**, at the application entry point. Libraries only call `getLogger(__name__)`.
- Pass arguments instead of f-strings to the logging call (`logger.info("x=%s", x)`), so formatting is skipped when the level is off and log aggregators can group messages.
- For JSON logs in services, use `structlog` or a JSON formatter.

## Debugging

| Xcode / LLDB | Python |
|---|---|
| Breakpoint in the gutter | `breakpoint()` in code, or a breakpoint in VS Code / PyCharm |
| `po value` | `p value` / `pp value` in pdb |
| Step over / into / out | `n` / `s` / `r` |
| Continue | `c` |
| `bt` | `w` (where) |
| Instruments Time Profiler | `python -m cProfile`, `py-spy`, 3.15 `profiling` |
| Address Sanitizer / Thread Sanitizer | `python -X dev`, `faulthandler` (no thread sanitizer) |

```python
def compute(values: list[int]) -> float:
    breakpoint()          # drops into pdb (respects PYTHONBREAKPOINT=0 to disable)
    return sum(values) / len(values)
```

| pdb command | Action |
|---|---|
| `n` / `s` | next line / step into |
| `c` | continue |
| `l`, `ll` | list source around / whole function |
| `p expr`, `pp expr` | print / pretty-print |
| `w` | stack trace (where) |
| `u` / `d` | move up/down the stack |
| `b file:line` | set breakpoint |
| `interact` | full Python REPL in the current frame |

`python -m pdb script.py` starts under the debugger; `python -X dev` enables extra runtime checks and warnings; `python -m cProfile -s cumtime script.py` profiles (3.15 adds a dedicated `profiling` package with a sampling profiler).

## Common errors decoded

| Error | Usual cause |
|---|---|
| `TypeError: 'NoneType' object is not subscriptable` | A function returned `None` (e.g. `list.sort()`), or a lookup failed silently |
| `AttributeError: 'NoneType' object has no attribute 'x'` | Same — `None` flowed further than expected |
| `UnboundLocalError` | Assigned to a name inside a function that you meant to read from an outer scope |
| `KeyError` | Use `.get()` or check `in` first |
| `IndexError: list index out of range` | Off-by-one; remember `range`/slices exclude the end |
| `ValueError: too many values to unpack` | Tuple/sequence length mismatch |
| `ModuleNotFoundError` | Wrong venv active, or running a file inside a package as a script (use `python -m pkg.module`) |
| `RecursionError` | Missing base case, or recursion that should be a loop |
| `RuntimeError: dictionary changed size during iteration` | Iterate over `list(d)` or build a new dict |

---

[← 05. Python Type System](05-type-system.md) · [Contents](../README.md) · [07. Modules, Packages and Dependency Management →](07-modules-and-packaging.md)
