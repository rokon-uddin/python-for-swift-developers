# Python Developer Reference: A Practical Guide for Experienced Software Engineers

A working reference for writing production Python — the language's actual semantics, the parts of the standard library worth knowing, concurrency, typing, data tooling, and the patterns I use when building data and LLM-backed services.

I came to Python after more than a decade of shipping native iOS apps in Swift and Objective-C. That background shapes this guide: it skips what any experienced engineer already knows (what a loop is, what a class is) and concentrates on where Python behaves differently from a compiled, strongly typed language — and where those differences cause real bugs.

**How to use it:** this page is the quick reference: vocabulary, the most useful Swift ↔ Python tables, and links to sixteen chapters in [`docs/`](docs/). Each chapter opens with an *In Swift terms* summary and shows Swift and Python side by side before going into Python-only details.

**Baseline:** Python 3.12+. Features from newer releases are marked (3.13, 3.14, 3.15). Every snippet is meant to be copy-pasteable.

> **Conventions used below**
> - `# → value` shows the result of an expression.
> - **Swift:** callouts compare against the Swift equivalent.
> - ⚠️ marks a pitfall worth remembering.
> - Side-by-side tables put the Swift you already know on the left and the Python equivalent on the right. Read the left column first, then spot what changed.

---

## Contents

- [Python Vocabulary for Swift Developers](#python-vocabulary-for-swift-developers)
- [Chapters](#chapters)
- [Swift ↔ Python at a Glance](#swift--python-at-a-glance)
- [Closing Notes](#closing-notes)
- [Official Documentation](#official-documentation)

---

## Python Vocabulary for Swift Developers

Python documentation uses its own words for ideas you already know. This table is the translation layer for the rest of the guide.

| Python term | Closest Swift idea | What it means in plain words |
|---|---|---|
| name / variable | `var` that always holds a **class** reference | A label stuck on an object. Assigning never copies |
| object | instance of a `class` | Every value — even `3` and `"hi"` — is an object |
| mutable / immutable | `var` struct vs `let` struct | A property of the **type** (list = mutable, tuple = immutable), not of the variable |
| `None` | `nil` | A real object. Any variable can hold it; there is no `Optional` wrapper to unwrap |
| `list` / `dict` / `set` / `tuple` | `Array` / `Dictionary` / `Set` / tuple | Same jobs, but lists, dicts and sets are **reference types** |
| iterable | `Sequence` | Something you can loop over |
| iterator | `IteratorProtocol` | The cursor doing the looping. Usually single-use |
| generator | a lazy `Sequence` you write with `yield` | A function that pauses after each value and resumes on the next request |
| comprehension | `map` + `filter` in one expression | `[x * 2 for x in xs if x > 0]` |
| decorator | a function that wraps a function (think property wrappers / macros applied to functions) | `@cache` above `def f` means `f = cache(f)` |
| dunder method (`__eq__`, `__len__`) | protocol requirement (`==` for `Equatable`, `count`) | "Double underscore" methods the language calls for you |
| duck typing | generics with no constraints that still compile | If an object has the method, you can call it — no conformance needed |
| `typing.Protocol` | `protocol`, but conformance is automatic | Structural typing: matching the shape is enough |
| type hints | type annotations with the compiler turned into a linter | Checked by separate tools (Pyright, mypy), ignored at runtime |
| module / package | a file / a module target | A `.py` file is a module; a folder of them is a package |
| virtual environment (venv) | per-project toolchain + `.build` dependencies | An isolated Python install for one project |
| pip / uv | Swift Package Manager | Installs and resolves dependencies |
| PEP | Swift Evolution proposal (SE-NNNN) | A numbered design document; PEP 8 is the style guide |
| `self` | `self` | Same idea, but written explicitly as the first parameter of every method |
| `__init__` | `init` | The initialiser |
| `@property` | computed property | Method that reads like an attribute |
| `@classmethod` / `@staticmethod` | `class func` / `static func` | Methods that don't need an instance |
| `with` block | `defer` + scope-based cleanup | Guarantees cleanup when the block exits |
| `raise` / `try` … `except` | `throw` / `do` … `catch` | Exceptions; nothing in the signature says a function can throw |
| EAFP | `try?` first, check later | "Easier to Ask Forgiveness than Permission": just try it and catch failure |
| LBYL | `guard` first | "Look Before You Leap": check conditions before acting |
| GIL | a global lock around the interpreter | In standard Python, only one thread runs Python code at any instant |
| event loop | an executor that runs all your async code on **one** thread | Like a world where every async function is `@MainActor` |
| coroutine | the value returned by calling an `async` function | Does nothing until it is awaited or scheduled as a task |
| MRO | — (Swift has single class inheritance) | Method Resolution Order: the order base classes are searched |
| LEGB | Swift's lexical scoping | Local → Enclosing → Global → Built-in: where Python looks up a name |

---

## Chapters

| # | Chapter | What's inside |
|---|---|---|
| 01 | [Python Language Fundamentals](docs/01-language-fundamentals.md) | Names vs values, mutability, `is` vs `==`, truthiness, numbers, strings, slicing, collections, comprehensions, iterators, control flow |
| 02 | [Swift vs Python](docs/02-swift-vs-python.md) | The semantic differences that cause bugs: constants, types, optionals, reference semantics, structs, protocols, closures, errors, enums, generics, memory |
| 03 | [Functions and Advanced Language Features](docs/03-functions.md) | Keyword arguments vs labels, defaults, scope, closures, decorators, generators, context managers, special methods |
| 04 | [Object-Oriented Programming](docs/04-oop.md) | Classes without structs: methods, properties, inheritance, ABCs, dataclasses, enums, access control |
| 05 | [Python Type System](docs/05-type-system.md) | Type hints, generics, `Protocol`, `TypedDict`, `Literal`, `Any` vs `object`, narrowing, Pyright/mypy |
| 06 | [Error Handling and Debugging](docs/06-errors-and-debugging.md) | Exceptions vs `throws`, custom errors, exception groups, logging, pdb vs LLDB |
| 07 | [Modules, Packages and Dependency Management](docs/07-modules-and-packaging.md) | Imports, packages, virtual environments, uv, `pyproject.toml` vs `Package.swift` |
| 08 | [Functional Programming and Pythonic Patterns](docs/08-functional-patterns.md) | `map`/`filter` chains → comprehensions, sorting, `collections`, `itertools`, `functools`, lazy pipelines |
| 09 | [File Handling and Data Serialization](docs/09-files-and-serialization.md) | `pathlib` vs `FileManager`, files, CSV, JSON vs `Codable`, environment variables |
| 10 | [Concurrency and Asynchronous Programming](docs/10-concurrency.md) | GIL, threads, processes, asyncio vs Swift Concurrency, task groups, cancellation, actors |
| 11 | [Testing and Code Quality](docs/11-testing-and-quality.md) | pytest vs Swift Testing, fixtures, mocking, Ruff, type checking |
| 12 | [Python Standard Library](docs/12-standard-library.md) | Foundation → standard library map, dates, regex, CLI parsing, subprocess, SQLite |
| 13 | [Python for Data Engineering and AI](docs/13-data-engineering.md) | NumPy, pandas, Polars, Arrow/Parquet, Spark, SQLAlchemy, Pydantic vs `Codable`, pipelines |
| 14 | [Python for LLM and AI Application Development](docs/14-llm-applications.md) | Clients, secrets, structured output, tool loops, retries, streaming, FastAPI vs Vapor, queues, RAG |
| 15 | [Common Pitfalls and Best Practices](docs/15-pitfalls.md) | The traps Swift developers fall into, and how to avoid them |
| 16 | [Practical Code Recipes](docs/16-recipes.md) | Copy-paste snippets for everyday tasks |

---

## Swift ↔ Python at a Glance

The tables most worth keeping open while you write Python. Each links to the chapter with the full explanation.

### Language and semantics

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

More: [Swift vs Python](docs/02-swift-vs-python.md)

### Control flow

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

More: [Language Fundamentals](docs/01-language-fundamentals.md#control-flow-compared)

### Collection operations

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

More: [Functional Patterns](docs/08-functional-patterns.md)

### Foundation → standard library

| Foundation / Swift | Python standard library |
|---|---|
| `Date`, `DateFormatter`, `ISO8601DateFormatter` | `datetime` |
| `TimeZone` | `zoneinfo.ZoneInfo` |
| `FileManager`, `URL` (file) | `pathlib`, `shutil`, `os` |
| `JSONEncoder` / `JSONDecoder` | `json` |
| `ProcessInfo.processInfo.environment` | `os.environ` |
| `CommandLine.arguments` | `sys.argv` |
| `Process` | `subprocess` |
| `Regex` / `NSRegularExpression` | `re` |
| `UUID()` | `uuid.uuid4()` |
| `URLComponents` | `urllib.parse` |
| `os.Logger` | `logging` |
| Swift Argument Parser (package) | `argparse` |
| `Set`, `Dictionary` helpers | `collections`, `itertools` |
| `DispatchQueue`, `OperationQueue` | `concurrent.futures`, `threading` |
| SQLite (C API / GRDB package) | `sqlite3` |
| `SystemRandomNumberGenerator` | `secrets` |

More: [Standard Library](docs/12-standard-library.md)

### Swift Concurrency → asyncio

| Swift | Python |
|---|---|
| `func load() async throws -> Data` | `async def load() -> bytes:` |
| `try await load()` | `await load()` |
| `Task { await load() }` | `asyncio.create_task(load())` (keep a reference) |
| `async let a = f(); async let b = g()` | `a, b = await asyncio.gather(f(), g())` |
| `withThrowingTaskGroup` | `async with asyncio.TaskGroup() as tg:` |
| `Task.sleep(for:)` | `await asyncio.sleep(seconds)` |
| `task.cancel()`, `Task.checkCancellation()` | `task.cancel()`; `CancelledError` raised at next `await` |
| `withTimeout` (custom) | `async with asyncio.timeout(s):` |
| `actor Counter { ... }` | Confine state to one task, or protect with `asyncio.Lock` |
| `@MainActor` | Nothing built in; the event loop thread is effectively the "main actor" |
| `AsyncStream` / `AsyncSequence` | Async generators / `asyncio.Queue` |
| `Sendable` | No equivalent — the compiler won't catch races |
| Runs on a multi-threaded cooperative pool | Runs on one thread; parallelism requires threads/processes |

More: [Concurrency](docs/10-concurrency.md#swift-concurrency--asyncio)

### Pitfalls

| Pitfall | What Swift does | What Python does |
|---|---|---|
| [Mutable default arguments](docs/15-pitfalls.md#mutable-default-arguments) | Re-evaluates defaults on each call | Evaluates once and reuses the object |
| [Shallow vs deep copying](docs/15-pitfalls.md#shallow-vs-deep-copying) | Nested arrays/structs copy as values | Copies are shallow; nested lists are shared |
| [Late binding in closures](docs/15-pitfalls.md#late-binding-in-closures) | Each `for` iteration has its own constant | One loop variable, read when the closure runs |
| [Variable scope](docs/15-pitfalls.md#variable-scope) | Assignment to an outer `var` just works | Assignment creates a local unless `nonlocal`/`global` |
| [Floating-point precision](docs/15-pitfalls.md#floating-point-precision) | Same IEEE doubles; `Decimal` in Foundation | Same; plus `round()` uses banker's rounding |
| [Iterator exhaustion](docs/15-pitfalls.md#iterator-exhaustion) | Arrays re-iterate; iterators are explicit | Generators silently produce nothing the second time |
| [Modifying while iterating](docs/15-pitfalls.md#modifying-a-collection-while-iterating) | You iterate a copy (value semantics) | `RuntimeError` for dicts, skipped items for lists |
| [Blocking in asyncio](docs/15-pitfalls.md#blocking-operations-in-asyncio) | Blocking hurts one pool thread | Blocking freezes every coroutine |
| [Exception anti-patterns](docs/15-pitfalls.md#exception-handling-anti-patterns) | Compiler forces `try` at throwing calls | Errors are invisible in signatures and easy to swallow |

More: [Pitfalls](docs/15-pitfalls.md)

---

## Closing Notes

Most of what makes Python feel loose to someone coming from Swift goes away with a few habits: type hints plus a strict checker, Pydantic at every boundary, dataclasses instead of loose dicts internally, `with` for every resource, explicit timeouts, and a clear choice between asyncio, threads and processes made up front rather than discovered in production.

What remains is what Python is genuinely good at: fast iteration, an enormous data and AI ecosystem, and code that is short enough to read in one sitting. This document will keep growing as I hit new edge cases — the pitfalls section most of all.

---

## Official Documentation

- [Python documentation (3.x)](https://docs.python.org/3/) · [What's New](https://docs.python.org/3/whatsnew/index.html) · [Language reference](https://docs.python.org/3/reference/index.html) · [Standard library](https://docs.python.org/3/library/index.html)
- [PEP 8 – Style Guide](https://peps.python.org/pep-0008/) · [PEP 20 – The Zen of Python](https://peps.python.org/pep-0020/) · [PEP 703 – Making the GIL optional](https://peps.python.org/pep-0703/)
- [`typing` module](https://docs.python.org/3/library/typing.html) · [Typing specification](https://typing.python.org/en/latest/spec/) · [mypy](https://mypy.readthedocs.io/) · [Pyright](https://microsoft.github.io/pyright/)
- [`asyncio`](https://docs.python.org/3/library/asyncio.html) · [`concurrent.futures`](https://docs.python.org/3/library/concurrent.futures.html) · [`threading`](https://docs.python.org/3/library/threading.html) · [`multiprocessing`](https://docs.python.org/3/library/multiprocessing.html)
- [Python Packaging User Guide](https://packaging.python.org/) · [`pyproject.toml` specification](https://packaging.python.org/en/latest/specifications/pyproject-toml/) · [uv](https://docs.astral.sh/uv/) · [Ruff](https://docs.astral.sh/ruff/)
- [pytest](https://docs.pytest.org/) · [`unittest.mock`](https://docs.python.org/3/library/unittest.mock.html)
- [NumPy](https://numpy.org/doc/stable/) · [pandas](https://pandas.pydata.org/docs/) · [Polars](https://docs.pola.rs/) · [Apache Arrow (Python)](https://arrow.apache.org/docs/python/) · [PySpark](https://spark.apache.org/docs/latest/api/python/) · [SQLAlchemy 2.0](https://docs.sqlalchemy.org/en/20/)
- [Pydantic](https://docs.pydantic.dev/latest/) · [pydantic-settings](https://docs.pydantic.dev/latest/concepts/pydantic_settings/) · [FastAPI](https://fastapi.tiangolo.com/) · [HTTPX](https://www.python-httpx.org/) · [Requests](https://requests.readthedocs.io/) · [Tenacity](https://tenacity.readthedocs.io/)
- [OpenAI Python SDK](https://github.com/openai/openai-python) · [LangChain](https://python.langchain.com/) · [LangGraph](https://langchain-ai.github.io/langgraph/) · [RQ](https://python-rq.org/) · [Hugging Face Transformers](https://huggingface.co/docs/transformers/)
