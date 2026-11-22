# 05. Python Type System

[← 04. Object-Oriented Programming](04-oop.md) · [Contents](../README.md) · [06. Error Handling and Debugging →](06-errors-and-debugging.md)

> **In Swift terms:** the annotations look a lot like Swift's, but they're checked by a separate program (Pyright or mypy), never by the interpreter. Treat the type checker as "the Swift compiler for Python" and run it in your editor and CI.

Type hints are optional, ignored at runtime, and enormously valuable.

## Built-in generics (no `typing` import needed)

| Swift | Python annotation |
|---|---|
| `[String]` | `list[str]` |
| `[String: Double]` | `dict[str, float]` |
| `Set<Int>` | `set[int]` |
| `(Double, Double)` | `tuple[float, float]` |
| `Int?` | `int \| None` |
| `(String, Int) -> Bool` | `Callable[[str, int], bool]` |
| `some Sequence<Double>` (parameter) | `Iterable[float]` |
| `Any` | `object` (see [`Any` vs `object`](#any-vs-object)) |
| `Void` | `None` |
| `Never` | `NoReturn` / `Never` |

```python
names: list[str] = []
scores: dict[str, float] = {}
point: tuple[float, float] = (0.0, 0.0)
record: tuple[int, ...] = (1, 2, 3)       # variable-length homogeneous tuple
unique: set[int] = set()
maybe: int | None = None                  # replaces Optional[int] (3.10+)
id_or_name: int | str = 1                 # replaces Union[int, str]
```

Abstract collection types for parameters live in `collections.abc` — accept the most general type you need, return a concrete one:

```python
from collections.abc import Iterable, Mapping, Sequence, Callable, Iterator

def total(values: Iterable[float]) -> float: return sum(values)
def lookup(table: Mapping[str, int], key: str) -> int: return table[key]
Handler = Callable[[str, int], bool]
```

## Type aliases and `NewType`

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
typealias UserID = Int

struct OrderID: Hashable {      // distinct type, no mixing
    let rawValue: Int
}
let oid = OrderID(rawValue: 42)
```

</td>
<td>

```python
from typing import NewType

type UserID = int                  # 3.12+ alias

OrderID = NewType("OrderID", int)  # distinct for the checker only
oid = OrderID(42)                  # still a plain int at runtime
```

</td>
</tr>
</table>

Type aliases can be recursive, which is handy for JSON:

```python
type JSON = dict[str, "JSON"] | list["JSON"] | str | int | float | bool | None
```

`NewType` gives some of the safety of a Swift wrapper struct with zero runtime cost.

## Generics: `TypeVar`, `Generic`, and 3.12 syntax

The Swift-vs-Python basics (generic functions, generic classes, `T: Hashable` bounds) are in [Swift vs Python → Generics](02-swift-vs-python.md#generics). Two extras:

```python
from typing import TypeVar, Generic

# Constrained: S must be exactly str or bytes (no Swift equivalent)
def longest[S: (str, bytes)](a: S, b: S) -> S:
    return a if len(a) >= len(b) else b

# Pre-3.12 spelling you'll still see in most codebases
T = TypeVar("T")
class Box(Generic[T]):
    def __init__(self, value: T) -> None:
        self.value = value
```

`ParamSpec` (shown in the decorator example) preserves a wrapped function's full signature.

## `Protocol`

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
protocol SupportsSummary {
    func summary() -> String
}

struct Report: SupportsSummary {
    func summary() -> String { "ok" }
}

let value: Any = Report()
if let s = value as? SupportsSummary {
    print(s.summary())
}
```

</td>
<td>

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class SupportsSummary(Protocol):
    def summary(self) -> str: ...

class Report:                       # no conformance declared
    def summary(self) -> str:
        return "ok"

value: object = Report()
if isinstance(value, SupportsSummary):   # ≈ `as?` with a protocol
    print(value.summary())               # → ok
```

</td>
</tr>
</table>

Structural interfaces. Add `@runtime_checkable` if you need `isinstance()` (checks only method presence, not signatures).

## `TypedDict`

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
struct Message: Codable {
    let role: String
    let content: String
    let name: String?            // optional key
}
```

</td>
<td>

```python
from typing import TypedDict, NotRequired

class Message(TypedDict):
    role: str
    content: str
    name: NotRequired[str]       # key may be missing

msg: Message = {"role": "user", "content": "hi"}
print(msg["role"])               # it IS a dict
```

</td>
</tr>
</table>

A `TypedDict` describes a dictionary's keys for the type checker; it doesn't decode or validate anything. For `Codable`-style decoding with validation, use Pydantic (section 13).

Types for dicts with a fixed set of string keys — ideal for JSON payloads you don't want to wrap in classes.

A `TypedDict` is still a plain `dict` at runtime — no validation. If the data comes from outside, validate with Pydantic (which accepts `TypedDict` as a type).

## `Literal`, `Final`, `Annotated`, `Self`, `Never`

```python
from typing import Literal, Final, Annotated, Self

Step = Literal["START", "PLAN", "TOOL", "OUTPUT"]

def route(step: Step) -> str: return step.lower()

TIMEOUT: Final = 30

class Builder:
    def with_name(self, name: str) -> Self:   # correct return type in subclasses too
        self.name = name
        return self

Port = Annotated[int, "1-65535"]   # metadata for tools/frameworks (FastAPI, Pydantic use it)
```

## `Any` vs `object`

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
let anything: Any = 42
// anything + 1            ← compile error: must cast first
if let n = anything as? Int {
    print(n + 1)
}
```

</td>
<td>

```python
from typing import Any

boxed: object = 42
# boxed + 1                ← type checker error (like Swift's Any)
if isinstance(boxed, int):
    print(boxed + 1)

unchecked: Any = 42
# unchecked.whatever()     # type checker allows ANYTHING (fails only at runtime)
```

</td>
</tr>
</table>

⚠️ Naming trap: Swift's `Any` behaves like Python's **`object`**. Python's `Any` means "turn type checking off for this value".

- `Any` turns type checking **off**: every operation is allowed. It spreads silently.
- `object` is the top type: anything is assignable to it, but you can't do anything with it without narrowing.

Use `object` for "accepts anything" parameters and `Any` only at untyped library edges.

## Narrowing

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
func length(_ value: Any?) -> Int {
    guard let value else { return 0 }
    if let s = value as? String { return s.count }
    if let a = value as? [Int] { return a.count }
    return 0
}
```

</td>
<td>

```python
def length(value: str | list[int] | None) -> int:
    if value is None:
        return 0
    if isinstance(value, str):    # value is `str` inside this branch
        return len(value)
    return len(value)             # checker knows: list[int]
```

</td>
</tr>
</table>

Python has no `as?` casts. `isinstance` checks (and `is None`) narrow the type inside the branch, the same way `if let` / `as?` does.

Checkers narrow types through `isinstance`, `is None`, `in`, `match`, and user-defined `TypeIs` (3.13+) / `TypeGuard` functions.

## Static checkers: Pyright and mypy

| | Pyright | mypy |
|---|---|---|
| Speed | Very fast (powers Pylance in VS Code) | Slower, incremental cache |
| Inference | Aggressive | More conservative |
| Config | `pyproject.toml` `[tool.pyright]` | `pyproject.toml` `[tool.mypy]` |

```toml
# pyproject.toml
[tool.pyright]
typeCheckingMode = "strict"
pythonVersion = "3.12"

[tool.mypy]
strict = true
python_version = "3.12"
```

Install stubs for untyped libraries (`types-requests`, `types-PyYAML`). Use `reveal_type(x)` while debugging annotations; checkers print the inferred type.

⚠️ Annotations are evaluated lazily in 3.14+ (PEP 649), so forward references no longer need quotes there. On 3.12/3.13 use string annotations (`"Order"`) or `from __future__ import annotations`.

---

[← 04. Object-Oriented Programming](04-oop.md) · [Contents](../README.md) · [06. Error Handling and Debugging →](06-errors-and-debugging.md)
