# 01. Python Language Fundamentals

[Contents](../README.md) · [02. Swift vs Python →](02-swift-vs-python.md)

> **In Swift terms:** picture Swift where every type is a `class`, `let` doesn't exist, the compiler never checks types, and `nil` can show up anywhere. Most of this section follows from those four differences.

## Syntax and conventions

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
let maxRetries = 3

/// Fetch a user.
func fetchUser(id: Int) -> [String: String] {
    if id < 0 {
        return [:]
    }
    return ["id": String(id)]   // braces end blocks
}
```

</td>
<td>

```python
MAX_RETRIES = 3

def fetch_user(user_id: int) -> dict[str, str]:
    """Fetch a user."""
    if user_id < 0:
        return {}
    return {"id": str(user_id)}   # indentation ends blocks
```

</td>
</tr>
</table>

No braces, no semicolons, no `func`/`let`/`var`. A colon opens a block; indentation closes it. Names are `snake_case` instead of `camelCase`.

- Blocks are defined by indentation — **4 spaces**, never tabs (PEP 8).
- `snake_case` for functions, variables and modules; `PascalCase` for classes; `UPPER_SNAKE` for module-level constants.
- A leading underscore (`_internal`) means "private by convention". Nothing enforces it.
- Lines max out around 88 (Black/Ruff default) or 79 (strict PEP 8) characters.
- `import this` prints the Zen of Python. The line that matters most in practice: *explicit is better than implicit*.

## Everything is an object; names are references

This is the single most important mental model. A variable is a **name bound to an object**, not a box holding a value. Every object has an **identity** (`id()`), a **type** (`type()`), and a **value**.

In Swift, `Array` is a struct, so assigning it makes an independent copy. In Python, `list` behaves like a Swift `class`: assigning it gives you a second name for the same object.

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
var a = [1, 2, 3]
var b = a          // independent copy (value type)
b.append(4)
print(a)           // [1, 2, 3]

final class Box { var items = [1, 2, 3] }
let x = Box()
let y = x          // same object (reference type)
y.items.append(4)
print(x.items)     // [1, 2, 3, 4]
```

</td>
<td>

```python
a = [1, 2, 3]
b = a              # same object — like the Box example
b.append(4)
print(a)           # → [1, 2, 3, 4]

c = a.copy()       # explicit copy = Swift's default behaviour
c.append(5)
print(a)           # → [1, 2, 3, 4]

x = 10
y = x
y += 1             # ints are immutable: += makes a NEW int
print(x, y)        # → 10 11
```

</td>
</tr>
</table>

Rule of thumb: in Python, **every** variable behaves like a Swift variable holding a class instance. Immutable types (`int`, `str`, `tuple`) only *look* like value types because nothing can change them in place.

Assignment never copies. Passing an argument never copies. This is "call by object reference" (sometimes called call-by-sharing).

## Mutable vs immutable

| Immutable | Mutable |
|---|---|
| `int`, `float`, `complex`, `bool`, `str`, `bytes`, `tuple`, `frozenset`, `None`, `frozendict` (3.15+) | `list`, `dict`, `set`, `bytearray`, most user-defined objects |

Operations on immutables that look like mutation (`s += "x"`, `n += 1`) create a new object and rebind the name. You can watch it happen:

```python
n = 2
before = id(n)
n = 12
print(before == id(n))   # → False: a different object

tags = set()
before = id(tags)
tags.add("python")
print(before == id(tags))  # → True: mutated in place
```

⚠️ A tuple is immutable, but the objects inside it are not: `t = ([], 1); t[0].append("x")` works.

## Identity vs equality: `is` vs `==`

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
final class Session {}
let s1 = Session(), s2 = s1

s1 === s2            // true  — same instance
[1, 2] == [1, 2]     // true  — equal values

let token: String? = nil
if token == nil { }
```

</td>
<td>

```python
class Session: ...
s1 = Session(); s2 = s1

print(s1 is s2)            # → True  (Swift ===)
print([1, 2] == [1, 2])    # → True

token: str | None = None
if token is None: ...      # None checks use `is`
```

</td>
</tr>
</table>

`is` is Swift's `===`; `==` is `==`. The one Python habit to build: compare with `None` using `is None`, never `== None`.

- `==` calls `__eq__` — compares **values**.
- `is` compares **identity** — "same object in memory".

Use `is` only for singletons: `None`, `True`, `False`, sentinel objects, and `...`. Never use `is` to compare numbers or strings — CPython caches small ints (−5…256) and interns some strings, so `is` *sometimes* appears to work and then fails in production.

```python
if value is None: ...        # correct
if value == None: ...        # works, but can be fooled by a custom __eq__
```

## Truthiness

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
let items: [String] = []
if items.isEmpty { print("empty") }

let input = ""
let name = input.isEmpty ? "anonymous" : input

let count = 0
// if count { }  ← compile error: Int is not Bool
```

</td>
<td>

```python
items: list[str] = []
if not items:
    print("empty")

user_input = ""
name = user_input or "anonymous"

count = 0
if not count:          # allowed: 0 is "falsy"
    print("zero")
```

</td>
</tr>
</table>

Swift's `if` only accepts a `Bool`. Python's `if` accepts **any** value and decides whether it counts as true. Empty and zero values count as false.

Falsy: `None`, `False`, `0`, `0.0`, `0j`, `""`, `b""`, and any empty container (`[]`, `{}`, `set()`, `()`, `range(0)`). Everything else is truthy unless it defines `__bool__` or `__len__`.

⚠️ `if not value` treats `0` and `""` the same as `None`. When `0` is a valid value, write `if value is None`.

`and` / `or` return **one of their operands**, not a `bool`:

```python
name = "" or "anonymous"      # → 'anonymous'
config: dict[str, int] = {}
port = config.get("port") or 8080   # → 8080 (but a configured port of 0 would also become 8080)
```

`bool` is a subclass of `int`: `True + True == 2`. Occasionally handy (`sum(flag for flag in flags)`), occasionally a bug (`isinstance(True, int)` is `True`).

## Numbers

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
import Foundation

7 / 2          // 3     Int / Int stays Int
7.0 / 2        // 3.5
-7 / 2         // -3    truncates toward zero
-7 % 3         // -1
// Int.max + 1 → overflow: compile error or runtime trap
pow(2.0, 10.0) // 1024.0 (Foundation)
Double(7) / 2  // explicit conversion required
```

</td>
<td>

```python
7 / 2          # 3.5   `/` ALWAYS returns float
7 // 2         # 3     floor division is a separate operator
-7 // 2        # -4    floors toward negative infinity
-7 % 3         # 2     result takes the divisor's sign
2 ** 100       # fine: ints never overflow
2 ** 10        # 1024
7 / 2.0        # mixing int and float just works
```

</td>
</tr>
</table>

Two traps for Swift developers: `/` on two ints gives a float, and integer division/modulo of negative numbers rounds differently.

- `float` is IEEE-754 double. `0.1 + 0.2 != 0.3`. Use `math.isclose()` for comparisons, `decimal.Decimal` for money, `fractions.Fraction` for exact rationals. `sys.float_info` shows the limits.

```python
from decimal import Decimal
import math

print(0.1 + 0.2)                          # → 0.30000000000000004
print(math.isclose(0.1 + 0.2, 0.3))       # → True
print(Decimal("0.1") + Decimal("0.2"))    # → 0.3   (construct from str, not float)
```

## Strings

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
import Foundation

let name = "Ada", balance = 1234.5
print("\(name) owes \(balance)")
print(String(format: "%.2f", balance))

"a,b,c".split(separator: ",")          // ["a", "b", "c"]
"  hi ".trimmingCharacters(in: .whitespaces)
["x", "y"].joined(separator: ", ")
"Swift".hasPrefix("Sw")
"swift".uppercased()
"café".count                           // 4 characters
```

</td>
<td>

```python
name, balance = "Ada", 1234.5
print(f"{name} owes {balance}")
print(f"{balance:.2f}")

"a,b,c".split(",")                     # ['a', 'b', 'c']
"  hi ".strip()
", ".join(["x", "y"])                  # join is called on the SEPARATOR
"Swift".startswith("Sw")
"swift".upper()
len("café")                            # 4 code points
```

</td>
</tr>
</table>

f-strings (`f"...{expr}..."`) are Python's string interpolation. Note `len(x)` instead of `x.count`, and `sep.join(list)` instead of `list.joined(separator:)`.

`str` is an immutable sequence of Unicode code points. `bytes` is an immutable sequence of integers 0–255. Converting between them is always explicit.

```python
label = "Café Spécial"
data = label.encode("utf-8")     # → b'Caf\xc3\xa9 Sp\xc3\xa9cial'
print(data.decode("utf-8"))      # → Café Spécial
print(len(label), len(data))     # → 12 14
```

f-strings are the default formatting tool. Since 3.12 they can reuse the outer quote type and contain any expression.

```python
user = {"name": "Ada", "balance": 1234.5}
print(f"{user["name"]} owes {user["balance"]:,.2f}")   # 3.12+: nested same quotes
print(f"{user['balance']=}")       # → user['balance']=1234.5   (debug specifier)
print(f"{42:08b} {255:#x} {0.256:.1%}")  # → 00101010 0xff 25.6%
print(f"{'left':<10}|{'right':>10}|")
```

Methods I reach for constantly: `strip/lstrip/rstrip`, `split/rsplit(sep, maxsplit)`, `join`, `replace`, `startswith/endswith` (accept a tuple), `removeprefix/removesuffix`, `casefold` (for case-insensitive comparison), `partition`, `isdigit/isalpha/isalnum`, `title`, `zfill`.

```python
" a, b ,c ".split(",")                  # → [' a', ' b ', 'c ']
[s.strip() for s in " a, b ,c ".split(",")]  # → ['a', 'b', 'c']
"report.final.csv".rsplit(".", 1)       # → ['report.final', 'csv']
", ".join(["x", "y"])                   # → 'x, y'
"v1.2.3".removeprefix("v")              # → '1.2.3'
```

⚠️ Building strings with `+=` in a loop is O(n²) in the worst case. Collect parts in a list and `"".join(parts)`.

## Slicing

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
let s = "Swift and Python"
s.prefix(5)                  // "Swift"
s.suffix(6)                  // "Python"
String(s.reversed())

let nums = [0, 1, 2, 3, 4, 5]
nums[1..<3]                  // ArraySlice [1, 2]
nums.dropFirst(2)            // [2, 3, 4, 5]
// nums[10...]               // crash: out of bounds
```

</td>
<td>

```python
s = "Swift and Python"
s[:5]                        # 'Swift'
s[-6:]                       # 'Python'  negative = from the end
s[::-1]                      # reversed

nums = [0, 1, 2, 3, 4, 5]
nums[1:3]                    # [1, 2]  a NEW list
nums[2:]                     # [2, 3, 4, 5]
nums[10:]                    # []  slices never crash
```

</td>
</tr>
</table>

`seq[start:stop:step]` works the same on strings, lists and tuples. Read `1:3` as Swift's `1..<3`. Omitted bounds mean "from the start" / "to the end".

`seq[start:stop:step]` — `stop` is exclusive, negative indices count from the end, and out-of-range slices never raise.

Two more slice tricks: `nums[1:3] = [10, 20, 30]` replaces a range (and can change the length), and `nums[:]` makes a shallow copy.

**Swift:** there is no `ArraySlice` type — slicing a list returns a new list (a copy); slicing a `str` returns a new `str`. For zero-copy views over binary data use `memoryview`; for arrays, NumPy slices are views.

## Unpacking

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
var (x, y) = (1, 2)
(x, y) = (y, x)                       // swap

func report() -> (sold: Int, left: Int) { (100, 20) }
let (sold, _) = report()

let values = [1, 2, 3, 4, 5]
let first = values.first!
let rest = Array(values.dropFirst())  // no one-line syntax
```

</td>
<td>

```python
x, y = 1, 2
x, y = y, x                           # swap

def report() -> tuple[int, int]:
    return 100, 20
sold, _ = report()

values = [1, 2, 3, 4, 5]
first, *rest = values                 # rest → [2, 3, 4, 5]
first, *middle, last = values         # middle → [2, 3, 4]
```

</td>
</tr>
</table>

Python's tuple destructuring works like Swift's, plus `*name` to collect "everything else" into a list.

Beyond the Swift-style cases above:

```python
(a, b), c = (1, 2), 3                           # nested patterns
for index, (key, value) in enumerate({"a": 1}.items()): ...
merged = {**{"a": 1}, **{"b": 2}}               # unpack dicts into a new dict
combined = [*range(3), *"ab"]                   # → [0, 1, 2, 'a', 'b']
```

⚠️ Unpacking needs the exact count: `a, b = [1, 2, 3]` raises `ValueError: too many values to unpack`.

## Core collections — what's different

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
var langs = ["swift", "kotlin"]
langs.append("python")
langs.insert("c", at: 0)
langs.removeAll { $0 == "kotlin" }
langs.sort()
let last = langs.removeLast()

var order = ["type": "latte"]
order["size"] = "L"
let sugar = order["sugar"]           // nil — no crash
let size = order["size", default: "M"]
order["size"] = nil                  // remove key

let a: Set = ["ios", "android"]
let b: Set = ["android", "web"]
a.union(b); a.intersection(b); a.subtracting(b)
```

</td>
<td>

```python
langs = ["swift", "kotlin"]
langs.append("python")
langs.insert(0, "c")                 # index comes first
langs.remove("kotlin")               # removes first match by value
langs.sort()
last = langs.pop()

order = {"type": "latte"}
order["size"] = "L"
sugar = order.get("sugar")           # None — order["sugar"] would RAISE KeyError
size = order.get("size", "M")
del order["size"]                    # or order.pop("size")

a = {"ios", "android"}
b = {"android", "web"}
a | b, a & b, a - b                  # union, intersection, difference
```

</td>
</tr>
</table>

The big one: `dict[key]` with a missing key **raises `KeyError`**, where Swift returns `nil`. Use `.get(key)` when the key may be absent.

A few things the table above doesn't show:

```python
items = ["c", "swift", "python"]
items.extend(["go", "rust"])         # append many (Swift: append(contentsOf:))
items.pop(0)                         # O(n) — use collections.deque for queues
sorted(items, reverse=True)          # new list; items.sort() sorts in place and returns None
grid = [0] * 3                       # → [0, 0, 0]

groups: dict[str, list[str]] = {}
groups.setdefault("extras", []).append("oat")   # get-or-insert in one call

a, b = {"ios", "android"}, {"android", "web"}
a ^ b                                # symmetric difference
frozen = frozenset(a)                # hashable → usable as a dict key or set member
```

- **Dicts preserve insertion order** (guaranteed since 3.7). `popitem()` removes the last inserted pair.
- Dict keys and set members must be **hashable** — immutable built-ins, tuples of hashables, frozen dataclasses.
- Membership (`in`) is O(1) for `set`/`dict`, O(n) for `list`/`tuple`.

## Comprehensions

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
let frameworks = ["SwiftUI", "UIKit", "FastAPI"]

let swiftish = frameworks.filter { $0.hasPrefix("Swift") }
let upper = frameworks.map { $0.uppercased() }
let lengths = Dictionary(uniqueKeysWithValues:
    frameworks.map { ($0, $0.count) })
let total = frameworks.map(\.count).reduce(0, +)
let flat = [[1, 2], [3]].flatMap { $0 }
```

</td>
<td>

```python
frameworks = ["SwiftUI", "UIKit", "FastAPI"]

swiftish = [f for f in frameworks if f.startswith("Swift")]
upper = [f.upper() for f in frameworks]
lengths = {f: len(f) for f in frameworks}

total = sum(len(f) for f in frameworks)
flat = [x for row in [[1, 2], [3]] for x in row]
```

</td>
</tr>
</table>

Read a comprehension as `map` (the expression on the left) + `filter` (the `if` on the right), written in one bracket. Square brackets build a list, curly braces a dict or set, parentheses a lazy generator.

```python
teams = {"backend": ["python", "go"], "mobile": ["swift", "kotlin", "python"]}
languages = {lang for stack in teams.values() for lang in stack}   # set comprehension
grid = [[0] * 3 for _ in range(3)]             # correct way to build a 2D list
```

Read nested `for` clauses left to right, exactly as if they were nested loops. If a comprehension needs more than two clauses or a side effect, write a loop.

⚠️ `[[0] * 3] * 3` creates **three references to the same inner list**.

## Iterables and iterators

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
let numbers = [1, 2, 3]
var it = numbers.makeIterator()
it.next()            // Optional(1)
it.next()            // Optional(2)
it.next()            // Optional(3)
it.next()            // nil — end

for n in numbers { }  // an Array can be looped many times
for n in numbers { }
```

</td>
<td>

```python
numbers = [1, 2, 3]
it = iter(numbers)
next(it)             # 1
next(it)             # 2
next(it)             # 3
next(it, None)       # None — without a default: raises StopIteration

squares = (n * n for n in numbers)
list(squares)        # [1, 4, 9]
list(squares)        # []  generators are single-use
```

</td>
</tr>
</table>

Swift signals the end with `nil`; Python raises `StopIteration` (the `for` loop catches it for you). Lists can be looped repeatedly; generators, files and most iterators only once.

- An **iterable** has `__iter__` (lists, dicts, files, generators, ranges).
- An **iterator** has `__next__` and raises `StopIteration` when done. It is single-pass.
- `for` calls `iter()` once, then `next()` until `StopIteration`.

`range`, `dict.keys()`, `dict.items()` are lazy *views*/sequences, not lists. `range(1, 11)` excludes 11 — upper bounds are exclusive everywhere in Python.

## Control flow compared

| Swift | Python |
|---|---|
| `for i in 0..<n` | `for i in range(n)` |
| `for i in 1...10` | `for i in range(1, 11)` — the end is always exclusive |
| `for i in stride(from: 0, to: 10, by: 2)` | `for i in range(0, 10, 2)` |
| `for (i, x) in xs.enumerated()` | `for i, x in enumerate(xs)` (add `start=1` to count from 1) |
| `for (a, b) in zip(xs, ys)` | `for a, b in zip(xs, ys)` |
| `for x in xs.reversed()` | `for x in reversed(xs)` |
| `repeat { … } while cond` | `while True: … if not cond: break` |
| `cond ? a : b` | `a if cond else b` |
| `if let x = f() { … }` | `if (x := f()) is not None: …` |
| `guard let x = f() else { return }` | `x = f()` then `if x is None: return` |
| `switch` | `match` (3.10+) |
| `0 <= x && x < 10` | `0 <= x < 10` (chained comparison) |
| `&&`, `\|\|`, `!` | `and`, `or`, `not` |

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
enum Event {
    case click(x: Int, y: Int)
    case key(Character)
}

func handle(_ event: Event) -> String {
    switch event {
    case let .click(x, y):
        return "click \(x),\(y)"
    case .key(let k) where k.isLetter:
        return "letter \(k)"
    default:
        return "other"
    }
}
```

</td>
<td>

```python
from dataclasses import dataclass

@dataclass
class Click:
    x: int
    y: int

@dataclass
class Key:
    char: str

def handle(event: Click | Key) -> str:
    match event:
        case Click(x=x, y=y):
            return f"click {x},{y}"
        case Key(char=k) if k.isalpha():
            return f"letter {k}"
        case _:
            return "other"

print(handle(Click(1, 2)))    # → click 1,2
```

</td>
</tr>
</table>

`match` is Python's `switch` with pattern matching. Each enum case with associated values becomes a small class; `case _:` is `default:`.

## Control-flow features without a direct Swift equivalent

```python
# for/else: the else block runs only if the loop did NOT `break`
# (Swift needs a `found` flag for this)
for name, age in [("Amit", 21), ("Zara", 17)]:
    if age < 18:
        print(f"{name} is a minor")
        break
else:
    print("everyone is an adult")

# Walrus in a loop condition: read-until-sentinel in one line
# while (line := input("> ")) != "quit":
#     print(line)
```

`match` can also destructure dictionaries and use `|` alternatives and `as` bindings — handy for JSON-shaped data:

```python
def handle(event: dict) -> str:
    match event:
        case {"type": "click", "pos": (x, y)}:
            return f"click at {x},{y}"
        case {"type": "key", "key": str(k)} if k.isalpha():
            return f"letter {k}"
        case {"type": "ac" | "sleeper" as seat}:
            return f"seat {seat}"
        case _:
            return "unknown"
```

⚠️ A bare name in a `case` **captures**; it does not compare. `case MAX:` binds any value to `MAX`. Compare against constants with dotted names (`case Status.OK:`) or literals.

For simple value → action mapping, a dispatch dict is often cleaner than `match` or an `if/elif` ladder:

```python
discounts = {"P20": (0.20, 0), "F10": (0.0, 10)}
percent, fixed = discounts.get("UNKNOWN", (0.0, 0))
```

---

[Contents](../README.md) · [02. Swift vs Python →](02-swift-vs-python.md)
