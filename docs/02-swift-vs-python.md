# 02. Swift vs Python

[← 01. Python Language Fundamentals](01-language-fundamentals.md) · [Contents](../README.md) · [03. Functions and Advanced Language Features →](03-functions.md)

> **In Swift terms:** this section is the cheat sheet. Each topic shows the Swift you would write, the Python you write instead, and the one behavioural difference most likely to cause a bug.

Syntax differences are trivial. The semantic ones below are what actually bite.

## Quick map

| Concept | Swift | Python |
|---|---|---|
| Constant | `let x = 1` | No real constants. `X: Final = 1` is a type-checker hint only |
| Variable | `var x = 1` | `x = 1` |
| Type checking | Compile time, mandatory | Runtime is dynamic; static checks are opt-in (mypy/Pyright) |
| Nullability | `String?`, must unwrap | `str \| None`; any name can be `None` at runtime |
| Value types | `struct`, `enum`, `Array`, `Dictionary` (copy-on-write) | None. Everything is a reference; immutables only *behave* like values |
| Array | `[Int]`, value semantics | `list[int]`, reference semantics |
| Protocol | Nominal, explicit conformance | `typing.Protocol` = structural; ABCs = nominal |
| Access control | `private`, `fileprivate`, `internal`, `public` | Conventions: `_name`, `__name` (name mangling) |
| Errors | `throws`, `try`, typed or untyped | Exceptions; any function can raise anything |
| Memory | ARC, retain cycles leak | Reference counting **plus** a cycle-collecting GC |
| Concurrency | async/await, actors, Sendable checked by compiler | `asyncio`, threads, processes; no data-race checking |
| Generics | Monomorphised/specialised, enforced | Erased; type parameters exist only for tools |
| Extensions | `extension Foo` | Monkey-patching is possible but discouraged; use functions or mixins |
| Overloading | By signature | No. One function per name; use defaults, `*args`, or `functools.singledispatch` |

## Variables and constants

Python has no `let`. `typing.Final` tells the type checker a name must not be rebound, but runtime ignores it. Immutability comes from **types** (tuple, frozenset, frozen dataclass), not from bindings.

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
let apiVersion = "v2"
// apiVersion = "v3"   ← compile error

var retries = 3
retries = 4

let tags = ["a"]
// tags.append("b")    ← compile error: `let` array
```

</td>
<td>

```python
from typing import Final

API_VERSION: Final = "v2"
API_VERSION = "v3"     # runs; only Pyright/mypy complain

retries = 3
retries = 4

tags = ("a",)          # immutability comes from the TYPE:
# tags.append("b")     # tuple has no append → AttributeError
```

</td>
</tr>
</table>

There is no `let`. To make data unchangeable, pick an immutable type: `tuple` instead of `list`, `frozenset` instead of `set`, a frozen dataclass instead of a class.

## Type system

Swift infers and enforces types at compile time. Python checks nothing at compile time; type hints are metadata. A function annotated `-> int` can return a string and the interpreter will not care.

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
func double(_ x: Int) -> Int {
    x * 2
}

// double("ab")   → compile error: String is not Int
```

</td>
<td>

```python
def double(x: int) -> int:
    return x * 2

double("ab")   # runs and returns 'abab'
               # Pyright/mypy report the error before you run it
```

</td>
</tr>
</table>

Annotations are documentation that tools can check. Run a type checker in your editor and CI to get back most of what the Swift compiler gives you.

I treat type hints as mandatory anyway and run Pyright or mypy in CI. That recovers most of what the Swift compiler gives you — minus the runtime guarantee. For runtime validation at system boundaries (HTTP bodies, LLM output, config) I use Pydantic.

## Optionals and `None`

There is no unwrapping. `None` is a regular singleton object and any variable can hold it. The type checker understands narrowing:

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
struct User { let name: String; let nickname: String? }

func displayName(_ user: User?) -> String {
    guard let user else { return "guest" }
    return user.nickname ?? user.name
}

// optional chaining: user?.nickname?.count
```

</td>
<td>

```python
from dataclasses import dataclass

@dataclass
class User:
    name: str
    nickname: str | None = None

def display_name(user: User | None) -> str:
    if user is None:
        return "guest"
    return user.nickname if user.nickname is not None else user.name

u = User("Ada")
length = len(u.nickname) if u.nickname is not None else None
print(display_name(u), display_name(None), length)  # → Ada guest None
```

</td>
</tr>
</table>

`str | None` is Python's `String?`, but nothing forces you to check it. Forget the `is None` check and you get an `AttributeError`/`TypeError` at runtime instead of a compile error.

| Swift | Python |
|---|---|
| `a ?? b` | `a if a is not None else b` (or `a or b` if falsy values are invalid) |
| `a?.b` | No operator. `a.b if a is not None else None`, or `getattr(a, "b", None)` |
| `if let x = f()` | `if (x := f()) is not None:` |
| `x!` | Just use `x`; it raises `AttributeError`/`TypeError` later if it was `None` |

## Arrays → lists (reference semantics)

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
func addItem(_ cart: inout [String]) {
    cart.append("book")
}

var myCart = ["pen"]
addItem(&myCart)      // mutation must be explicit
print(myCart)         // ["pen", "book"]

func withItem(_ cart: [String]) -> [String] {
    cart + ["book"]   // caller's array untouched
}
```

</td>
<td>

```python
def add_item(cart: list[str]) -> None:
    cart.append("book")          # no `inout` needed — or possible to prevent

my_cart = ["pen"]
add_item(my_cart)
print(my_cart)                   # → ['pen', 'book']

def with_item(cart: list[str]) -> list[str]:
    return cart + ["book"]       # builds a new list; caller untouched
```

</td>
</tr>
</table>

Every Python function that receives a list, dict or set can modify the caller's object. Convention: functions that mutate return `None`; functions that return a value don't mutate.

The most common Swift-brain bug:

```python
def add_item(cart: list[str]) -> None:
    cart.append("book")       # mutates the caller's list

my_cart = ["pen"]
add_item(my_cart)
print(my_cart)                # → ['pen', 'book']
```

In Swift the function would receive a copy (and could not mutate it without `inout`). In Python, mutate-in-place functions should say so in their name/docs, and conventionally return `None` (like `list.sort()`). Functions that produce a new value should not mutate their input.

```python
from copy import copy, deepcopy

nested = [[1, 2], [3]]
shallow = copy(nested)        # or nested[:] / list(nested)
deep = deepcopy(nested)
shallow[0].append(99)
print(nested[0])              # → [1, 2, 99]  (shared inner list)
print(deep[0])                # → [1, 2]
```

## Dictionaries and sets

Equivalent capabilities; different defaults:

- Accessing a missing key with `d[k]` raises `KeyError` (Swift returns `nil`). Use `d.get(k)`.
- Dicts are ordered by insertion; Swift's are not.
- `collections.defaultdict` and `Counter` cover most `Dictionary(grouping:)` / `reduce(into:)` uses.

## Structs and classes

Python has no value types. The closest thing to a Swift `struct` is a frozen dataclass with `slots`:

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
struct Point: Hashable {
    var x: Double
    var y: Double
}

let p = Point(x: 1, y: 2)
var q = p           // copy
q.x = 10
print(p, q, p == Point(x: 1, y: 2))
```

</td>
<td>

```python
from dataclasses import dataclass, replace

@dataclass(frozen=True, slots=True)
class Point:
    x: float
    y: float

p = Point(1, 2)
q = replace(p, x=10)   # "copy with changes"
print(p, q, p == Point(1, 2))
# → Point(x=1, y=2) Point(x=10, y=2) True
```

</td>
</tr>
</table>

A frozen dataclass gives you a struct's memberwise init, `==`, hashing and readable printing. It is still a reference, but because it can't change, sharing it is safe.

## Protocols and structural typing

Swift protocols are nominal: a type conforms only if it declares conformance. `typing.Protocol` is **structural** — anything with the right shape conforms, like Go interfaces or TypeScript types.

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
protocol Closable {
    func close()
}

final class Connection: Closable {   // must declare conformance
    func close() { print("closed") }
}

func shutdown(_ r: some Closable) {
    r.close()
}
shutdown(Connection())
```

</td>
<td>

```python
from typing import Protocol

class Closable(Protocol):
    def close(self) -> None: ...

class Connection:                     # never mentions Closable
    def close(self) -> None:
        print("closed")

def shutdown(resource: Closable) -> None:
    resource.close()

shutdown(Connection())                # → closed
```

</td>
</tr>
</table>

Conformance is automatic: any class with a matching `close()` method satisfies `Closable`. The type checker verifies it; at runtime nothing is checked.

Use `abc.ABC` when you want nominal conformance and a runtime error for unimplemented methods.

## Closures and lambdas

- Python closures capture **variables, not values** (late binding — see [pitfalls](15-pitfalls.md#late-binding-in-closures)).
- To assign to a captured variable you need `nonlocal` (or `global`). Swift closures can mutate captured `var`s freely.
- `lambda` is limited to a single expression. Anything longer is a nested `def` — which is completely normal in Python.
- No capture lists, no `[weak self]`: the garbage collector handles cycles.

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
func makeCounter() -> () -> Int {
    var count = 0
    return {
        count += 1          // can mutate captured var
        return count
    }
}

let counter = makeCounter()
_ = counter()
print(counter())            // 2

let doubled = [1, 2].map { $0 * 2 }
```

</td>
<td>

```python
def make_counter():
    count = 0
    def increment() -> int:
        nonlocal count      # required to reassign a captured variable
        count += 1
        return count
    return increment

counter = make_counter()
counter()
print(counter())            # → 2

doubled = list(map(lambda x: x * 2, [1, 2]))
```

</td>
</tr>
</table>

## Error handling

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
enum ParseError: Error { case invalid(String) }

func parse(_ raw: String) throws -> Int {
    guard let n = Int(raw) else {
        throw ParseError.invalid(raw)
    }
    return n
}

do {
    let n = try parse("4x")
    print(n)
} catch ParseError.invalid(let raw) {
    print("bad input: \(raw)")
}
let maybe = try? parse("4x")   // nil
```

</td>
<td>

```python
class ParseError(Exception):
    pass

def parse(raw: str) -> int:          # no `throws` in the signature
    try:
        return int(raw)
    except ValueError:
        raise ParseError(f"invalid: {raw}") from None

try:
    n = parse("4x")
    print(n)
except ParseError as exc:
    print(f"bad input: {exc}")       # → bad input: invalid: 4x

try:
    maybe = parse("4x")
except ParseError:
    maybe = None                     # Python has no `try?`
```

</td>
</tr>
</table>

Any Python function can raise any exception, and calls need no `try` keyword. Read docs (or the source) to know what a function raises.

| Swift | Python |
|---|---|
| `func f() throws` | Nothing in the signature. Document raised exceptions in the docstring |
| `do { try f() } catch E { }` | `try: f()` / `except E:` |
| `try?` | `contextlib.suppress(E)` or a `try/except` returning `None` |
| `defer` | `finally`, or a context manager (`with`) |
| `Result<T, E>` | Not idiomatic; exceptions are the norm |

Python's culture is **EAFP** ("easier to ask forgiveness than permission"): attempt the operation and catch the failure, rather than checking first (LBYL). Exceptions are cheap enough to be used for control flow (`StopIteration`, `KeyError`).

## Enums

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
enum Status: String, CaseIterable {
    case active, suspended
}
Status(rawValue: "active") == .active   // true
Status.allCases.count                   // 2

enum Shape {
    case circle(r: Double)
    case rect(w: Double, h: Double)
}

func area(_ s: Shape) -> Double {
    switch s {
    case .circle(let r): return .pi * r * r
    case let .rect(w, h): return w * h
    }
}
```

</td>
<td>

```python
from dataclasses import dataclass
from enum import StrEnum, auto

class Status(StrEnum):
    ACTIVE = auto()                       # value 'active'
    SUSPENDED = auto()
print(Status("active") is Status.ACTIVE)  # → True
print(len(Status))                        # → 2

@dataclass(frozen=True)
class Circle:
    r: float

@dataclass(frozen=True)
class Rect:
    w: float
    h: float

type Shape = Circle | Rect                # 3.12+

def area(s: Shape) -> float:
    match s:
        case Circle(r=r): return 3.14159 * r * r
        case Rect(w=w, h=h): return w * h

print(area(Rect(2, 3)))                   # → 6
```

</td>
</tr>
</table>

Simple enums map directly. Enums **with associated values** don't exist; model each case as a small class and combine them with `|`.

## Generics

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
func first<T>(_ items: [T]) -> T? {
    items.first
}

struct Stack<Element> {
    private var items: [Element] = []
    mutating func push(_ item: Element) { items.append(item) }
    mutating func pop() -> Element? { items.popLast() }
}

func unique<T: Hashable>(_ xs: [T]) -> Set<T> { Set(xs) }
```

</td>
<td>

```python
from collections.abc import Hashable

def first[T](items: list[T]) -> T | None:
    return items[0] if items else None

class Stack[T]:
    def __init__(self) -> None:
        self._items: list[T] = []
    def push(self, item: T) -> None:
        self._items.append(item)
    def pop(self) -> T | None:
        return self._items.pop() if self._items else None

def unique[T: Hashable](xs: list[T]) -> set[T]:
    return set(xs)
```

</td>
</tr>
</table>

Syntax is close to Swift since 3.12 (`[T]` instead of `<T>`). The difference: generics are only for tools. At runtime `Stack[int]` will happily hold strings.

Bounds use a colon (`[T: Hashable]`) where Swift uses `<T: Hashable>`; there is no `where` clause and no specialisation.

## Memory management

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
final class Screen {
    var onTap: (() -> Void)?
    func setup() {
        onTap = { [weak self] in   // avoid a retain cycle
            self?.refresh()
        }
    }
    func refresh() {}
}
```

</td>
<td>

```python
class Screen:
    def __init__(self) -> None:
        self.on_tap = None
    def setup(self) -> None:
        self.on_tap = lambda: self.refresh()   # cycle is fine:
    def refresh(self) -> None: ...             # the GC frees it

s = Screen(); s.setup(); del s                 # no leak
```

</td>
</tr>
</table>

Python counts references like ARC **and** runs a garbage collector that finds cycles. You don't write `weak`/`unowned` to prevent leaks; `weakref` exists only for caches and observers.

CPython uses **reference counting** (like ARC) for immediate deallocation, plus a **cyclic garbage collector** that finds and frees reference cycles. So the classic Swift retain-cycle leak (closure capturing `self`, parent ↔ child strong refs) does not leak in Python — it just gets freed later. `weakref` exists for caches and observer patterns where you explicitly don't want to keep objects alive.

You almost never think about memory in Python — until you load 10 GB into a list. Then the answer is generators, chunked processing, NumPy/Arrow buffers, or a smaller dtype.

## Concurrency

| Swift Concurrency | Python |
|---|---|
| `async func`, `await` | `async def`, `await` — very similar |
| `Task { }` | `asyncio.create_task(coro)` |
| `async let` / `withTaskGroup` | `asyncio.TaskGroup` (3.11+), `asyncio.gather` |
| Cooperative thread pool across cores | **Single-threaded** event loop |
| Actors, `@MainActor` | No actors. Confine state to one task, use `asyncio.Lock`/`Queue` |
| `Sendable` checking | None. Data races are your problem |
| Cancellation via `Task.cancel()` | `task.cancel()` raises `CancelledError` inside the coroutine |
| Real parallelism by default | Threads limited by the GIL (default builds); use processes for CPU work |

The key mental difference: Swift's `async` code runs on a multi-threaded cooperative pool. Python's `asyncio` runs every coroutine on **one thread**, so a single blocking call (`time.sleep`, `requests.get`, a CPU loop) freezes everything. Details in [section 10](10-concurrency.md).

---

[← 01. Python Language Fundamentals](01-language-fundamentals.md) · [Contents](../README.md) · [03. Functions and Advanced Language Features →](03-functions.md)
