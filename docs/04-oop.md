# 04. Object-Oriented Programming

[← 03. Functions and Advanced Language Features](03-functions.md) · [Contents](../README.md) · [05. Python Type System →](05-type-system.md)

> **In Swift terms:** Python only has classes — no structs, and no enums with associated values. Classes support multiple inheritance, nothing is truly private, and you can add attributes to an object at any time. Dataclasses fill the role of structs.

## Classes, instances, and namespaces

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
final class Server {
    static let scheme = "https"      // shared by all instances
    let port: Int                    // declared stored property

    init(port: Int) {
        self.port = port
    }

    func describe() -> String {
        "\(Server.scheme) server on :\(port)"
    }
}
```

</td>
<td>

```python
class Server:
    scheme = "https"                 # class attribute (≈ static let)

    def __init__(self, port: int) -> None:
        self.port = port             # stored property is created HERE

    def describe(self) -> str:       # `self` is an explicit parameter
        return f"{self.scheme} server on :{self.port}"

print(Server(8443).describe())       # → https server on :8443
```

</td>
</tr>
</table>

No property declarations: instance attributes come into existence when `__init__` assigns them. Every method lists `self` as its first parameter, and you must write `self.` to read attributes.

- `self` is explicit and is just the first parameter. `server.describe()` is `Server.describe(server)`.
- `__init__` is an **initialiser**, not a constructor. `__new__` creates the object; you rarely touch it.
- Classes are objects too: `type(Cup)` is `type`. You can add attributes to a class or instance at runtime.

## Attribute lookup and shadowing

Lookup order for `obj.attr`: instance `__dict__` → class → base classes (MRO). Assigning to `obj.attr` always writes to the **instance**, shadowing the class attribute. Deleting the instance attribute makes the class value visible again.

```python
class Settings:
    theme = "dark"

s = Settings()
s.theme = "light"          # instance attribute shadows the class one
print(Settings.theme)      # → dark
del s.theme
print(s.theme)             # → dark (falls back to class)
```

⚠️ A mutable class attribute is shared by every instance:

```python
class Bad:
    items = []             # shared!
class Good:
    def __init__(self) -> None:
        self.items: list[str] = []
```

## Instance, class, and static methods

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
final class Order {
    let kind: String, size: String
    init(kind: String, size: String) {
        self.kind = kind; self.size = size
    }

    static func from(string raw: String) -> Order {
        let parts = raw.split(separator: "-").map(String.init)
        return Order(kind: parts[0], size: parts[1])
    }

    static func isValid(size: String) -> Bool {
        ["S", "M", "L"].contains(size)
    }
}
```

</td>
<td>

```python
class Order:
    def __init__(self, kind: str, size: str) -> None:
        self.kind, self.size = kind, size

    @classmethod
    def from_string(cls, raw: str) -> "Order":   # factory
        kind, size = raw.split("-")
        return cls(kind, size)        # `cls` = the actual (sub)class

    @staticmethod
    def is_valid_size(size: str) -> bool:
        return size in {"S", "M", "L"}

print(Order.from_string("latte-S").kind)   # → latte
```

</td>
</tr>
</table>

Use `@classmethod` for alternative initialisers (Python has no `convenience init` or overloaded `init`). `@staticmethod` is Swift's `static func`, though a plain module-level function is often simpler.

| | Receives | Typical use |
|---|---|---|
| Instance method | `self` | Behaviour using instance state |
| `@classmethod` | `cls` | Factory / alternative constructors; respects subclasses (`cls(...)`) |
| `@staticmethod` | nothing | Namespaced helper. Often a module-level function is better |

**Swift:** `@classmethod` ≈ `class func` / `static func` factory; Python has no `convenience init` — factories are classmethods.

## Properties

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
final class Sensor {
    var celsius: Double {
        didSet { precondition(celsius >= -273.15) }
    }
    var fahrenheit: Double {          // computed, read-only
        celsius * 9 / 5 + 32
    }
    lazy var calibration = loadCalibration()

    init(celsius: Double) { self.celsius = celsius }
    func loadCalibration() -> [Double] { [] }
}
```

</td>
<td>

```python
from functools import cached_property

class Sensor:
    def __init__(self, celsius: float) -> None:
        self.celsius = celsius            # goes through the setter

    @property
    def celsius(self) -> float:
        return self._celsius

    @celsius.setter                       # ≈ willSet/didSet + validation
    def celsius(self, value: float) -> None:
        if value < -273.15:
            raise ValueError("below absolute zero")
        self._celsius = value

    @property
    def fahrenheit(self) -> float:        # computed, read-only
        return self._celsius * 9 / 5 + 32

    @cached_property                      # ≈ lazy var
    def calibration(self) -> list[float]:
        return []
```

</td>
</tr>
</table>

More ceremony than Swift, but you rarely need it: start with plain attributes and switch to `@property` later without changing any caller.

`@property` gives computed attributes and validated setters while keeping attribute syntax — the equivalent of Swift computed properties and `didSet`/`willSet`.

Start with plain attributes. You can turn one into a property later without changing callers — which is why Python code doesn't need getter/setter boilerplate up front.

`functools.cached_property` computes once and stores the result on the instance (like a `lazy var`).

## Inheritance and `super()`

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
class Vehicle {
    let name: String
    init(name: String) { self.name = name }
    func describe() -> String { name }
}

final class ElectricVehicle: Vehicle {
    let batteryKWh: Double
    init(name: String, batteryKWh: Double) {
        self.batteryKWh = batteryKWh
        super.init(name: name)
    }
    override func describe() -> String {
        super.describe() + " (EV)"
    }
}
```

</td>
<td>

```python
from typing import override

class Vehicle:
    def __init__(self, name: str) -> None:
        self.name = name
    def describe(self) -> str:
        return self.name

class ElectricVehicle(Vehicle):
    def __init__(self, name: str, battery_kwh: float) -> None:
        super().__init__(name)            # order doesn't matter here
        self.battery_kwh = battery_kwh

    @override                             # optional; checked by tools only
    def describe(self) -> str:
        return super().describe() + " (EV)"

print(ElectricVehicle("Model", 75).describe())   # → Model (EV)
```

</td>
</tr>
</table>

No `override` keyword is required and no two-phase initialisation: you can call `super().__init__` anywhere in `__init__` (or forget it — nothing stops you). Classes are open for subclassing by default; there is no `final` enforcement at runtime.

Three ways to reach the base class: duplicate the code (don't), call `Base.__init__(self, ...)` explicitly (breaks with multiple inheritance), or `super()` (cooperative, follows the MRO — use this).

Use `typing.override` (3.12+) so the type checker catches overrides that no longer match a base method.

## Composition over inheritance

Holding a collaborator as an attribute — and letting subclasses swap the collaborator *class* — is often cleaner than deep inheritance:

```python
class Formatter:
    def render(self) -> str: return "plain"
class JSONFormatter(Formatter):
    def render(self) -> str: return "json"

class Exporter:
    formatter_cls: type[Formatter] = Formatter
    def __init__(self) -> None:
        self.formatter = self.formatter_cls()  # has-a, not is-a
    def export(self) -> str:
        return self.formatter.render()

class APIExporter(Exporter):
    formatter_cls = JSONFormatter

print(Exporter().export(), APIExporter().export())   # → plain json
```

For real dependency injection, pass the collaborator into `__init__`.

## Multiple inheritance and MRO

Python supports multiple inheritance and resolves attribute lookup with the **C3 linearisation** (Method Resolution Order).

```python
class A: label = "A"
class B(A): label = "B"
class C(A): label = "C"
class D(C, B): pass

print(D().label)                          # → C
print([k.__name__ for k in D.__mro__])    # → ['D', 'C', 'B', 'A', 'object']
```

Rule of thumb: left-to-right in the base list, children before parents, each class once. `super()` means "next in the MRO", not "my parent" — that's what makes cooperative mixins work. I limit multiple inheritance to small, stateless mixins.

## Abstract base classes

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
protocol Repository {
    func get(_ key: String) -> [String: String]?
}

extension Repository {                       // shared default code
    func exists(_ key: String) -> Bool { get(key) != nil }
}

final class MemoryRepository: Repository {
    var data: [String: [String: String]] = [:]
    func get(_ key: String) -> [String: String]? { data[key] }
}
```

</td>
<td>

```python
from abc import ABC, abstractmethod

class Repository(ABC):
    @abstractmethod
    def get(self, key: str) -> dict | None: ...

    def exists(self, key: str) -> bool:      # shared default code
        return self.get(key) is not None

class MemoryRepository(Repository):
    def __init__(self) -> None:
        self.data: dict[str, dict] = {}
    def get(self, key: str) -> dict | None:
        return self.data.get(key)

print(MemoryRepository().exists("x"))       # → False
```

</td>
</tr>
</table>

An ABC is the closest thing to "protocol + protocol extension": abstract requirements plus shared implementation. The catch is that you must inherit from it, and the missing-method error only appears when you try to create an instance.

ABC = nominal (must subclass, enforced at instantiation). `Protocol` = structural (checked statically, no inheritance needed). I default to `Protocol` for dependency boundaries and ABCs when sharing implementation.

## Dataclasses

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
struct Ticket {
    let id: Int
    let title: String
    var tags: [String] = []
    var priority: Int = 3

    init(id: Int, title: String,
         tags: [String] = [], priority: Int = 3) {
        precondition((1...5).contains(priority))
        self.id = id; self.title = title
        self.tags = tags; self.priority = priority
    }
}
let t = Ticket(id: 1, title: "Fix login")
```

</td>
<td>

```python
from dataclasses import dataclass, field

@dataclass(kw_only=True)                  # labels required, like Swift
class Ticket:
    id: int
    title: str
    tags: list[str] = field(default_factory=list)
    priority: int = 3

    def __post_init__(self) -> None:      # runs after the generated init
        if not 1 <= self.priority <= 5:
            raise ValueError("priority must be 1..5")

t = Ticket(id=1, title="Fix login")
print(t)   # → Ticket(id=1, title='Fix login', tags=[], priority=3)
```

</td>
</tr>
</table>

`@dataclass` writes the memberwise initialiser, `==` and a readable printout for you, like a Swift struct. Validation that would go in a custom `init` goes in `__post_init__`.

`@dataclass` generates `__init__`, `__repr__`, `__eq__` (and optionally ordering, hashing, `__slots__`).

| Option | Effect |
|---|---|
| `frozen=True` | Immutable, hashable |
| `slots=True` | `__slots__`: less memory, faster attribute access, no ad-hoc attributes |
| `kw_only=True` | All fields keyword-only — readable call sites |
| `order=True` | Generates `<`, `<=`, … comparing fields as a tuple |

**Dataclass vs Pydantic:** dataclasses don't validate types at runtime. Use dataclasses for internal domain objects; Pydantic models at I/O boundaries.

## Enums

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
enum Priority: Int, Comparable {
    case low = 1, high = 2
    static func < (a: Self, b: Self) -> Bool {
        a.rawValue < b.rawValue
    }
}

struct Permission: OptionSet {
    let rawValue: Int
    static let read  = Permission(rawValue: 1 << 0)
    static let write = Permission(rawValue: 1 << 1)
    static let admin: Permission = [.read, .write]
}
Permission.admin.contains(.write)    // true
```

</td>
<td>

```python
from enum import IntEnum, Flag, auto

class Priority(IntEnum):     # comparable for free
    LOW = 1
    HIGH = 2

class Permission(Flag):      # ≈ OptionSet
    READ = auto()
    WRITE = auto()
    ADMIN = READ | WRITE

print(Priority.HIGH > Priority.LOW)          # → True
print(Permission.WRITE in Permission.ADMIN)  # → True
```

</td>
</tr>
</table>

Python enum members are written in `UPPER_CASE` and accessed as `Priority.HIGH` (no leading-dot shorthand).

## Encapsulation and access control

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
public final class Account {
    public let id: String
    private var balance: Double = 0
    fileprivate func audit() {}

    public init(id: String) { self.id = id }
}
// account.balance  ← compile error
```

</td>
<td>

```python
class Account:
    def __init__(self, id: str) -> None:
        self.id = id
        self._balance = 0.0     # "private by convention"
        self.__pin = "1234"     # name-mangled to _Account__pin

    def _audit(self) -> None: ...

acct = Account("a1")
print(acct._balance)            # allowed; linters warn
```

</td>
</tr>
</table>

Python's access control is a naming convention, not a compiler rule. A leading underscore tells other developers "internal, don't touch".

Python trusts the caller. The conventions:

| Swift | Python |
|---|---|
| `public` | `name` |
| `internal` / `fileprivate` | `_name` — "don't touch from outside"; excluded from `from m import *` |
| `private` | `__name` — name-mangled to `_ClassName__name`, avoids clashes in subclasses; still reachable |
| Module exports | `__all__ = ["PublicThing"]` |

## `__slots__`

By default each instance carries a `__dict__`. `__slots__` replaces it with fixed storage: less memory per object (significant when you have millions) and typos in attribute names raise `AttributeError` instead of silently creating a new attribute. `@dataclass(slots=True)` sets it up for you.

---

[← 03. Functions and Advanced Language Features](03-functions.md) · [Contents](../README.md) · [05. Python Type System →](05-type-system.md)
