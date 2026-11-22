# 12. Python Standard Library

[← 11. Testing and Code Quality](11-testing-and-quality.md) · [Contents](../README.md) · [13. Python for Data Engineering and AI →](13-data-engineering.md)

> **In Swift terms:** think of this as Foundation. Most things Foundation gives you (`Date`, `FileManager`, `JSONEncoder`, `Process`, `UUID`, regex, URL parsing) have a standard-library module here, plus several Swift needs packages for (CLI parsing, SQLite, CSV, statistics).

## Quick reference

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

| Module | Use it for |
|---|---|
| `os` | Environment variables, process info, low-level file ops (`os.replace`, `os.cpu_count`) |
| `sys` | Interpreter: `argv`, `path`, `exit`, `version_info`, `stdin/stdout`, `getsizeof` |
| `pathlib` | Object-oriented paths and file I/O |
| `datetime` | Dates, times, timezone-aware timestamps, `timedelta` |
| `zoneinfo` | IANA time zones (3.9+) |
| `collections` | `Counter`, `defaultdict`, `deque`, `namedtuple`, `ChainMap` |
| `itertools` | Lazy iterator building blocks |
| `functools` | `cache`, `partial`, `reduce`, `wraps`, `singledispatch`, `total_ordering` |
| `operator` | Functions for operators: `itemgetter`, `attrgetter`, `add`, `mul` |
| `math` | `isqrt`, `isclose`, `ceil`, `floor`, `prod`, `dist`, `inf` |
| `statistics` | `mean`, `median`, `stdev`, `quantiles`, `correlation`, `linear_regression` |
| `json`, `csv`, `tomllib` | Serialization |
| `re` | Regular expressions |
| `logging` | Application logging |
| `argparse` | CLI argument parsing |
| `subprocess` | Running external commands |
| `asyncio` | Event loop, coroutines, tasks, queues |
| `concurrent.futures` | Thread/process pools |
| `dataclasses`, `enum`, `typing` | Data modelling and types |
| `contextlib` | Context-manager utilities |
| `shutil`, `tempfile` | High-level file ops, temporary files/dirs |
| `hashlib`, `secrets`, `hmac` | Hashing, secure random tokens, signatures |
| `uuid` | Unique IDs |
| `sqlite3` | Embedded SQL database |
| `heapq`, `bisect` | Priority queues, binary search on sorted lists |
| `textwrap`, `string` | Text wrapping/dedent; character constants (`ascii_letters`, `punctuation`) |
| `copy` | `copy`/`deepcopy` |
| `urllib.parse` | URL parsing/joining/encoding (`urljoin`, `urlencode`, `quote`) |
| `timeit`, `cProfile` | Benchmarking and profiling |

## `datetime` and `zoneinfo`

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
import Foundation

let now = Date()
let tomorrow = Calendar.current.date(byAdding: .day, value: 1, to: now)!

let iso = ISO8601DateFormatter()
let stamp = iso.string(from: now)
let parsed = iso.date(from: "2026-11-21T09:30:00Z")

let f = DateFormatter()
f.dateFormat = "yyyy-MM-dd HH:mm"
f.timeZone = TimeZone(identifier: "Asia/Dhaka")
print(f.string(from: now))
```

</td>
<td>

```python
from datetime import datetime, timedelta, timezone
from zoneinfo import ZoneInfo

now = datetime.now(timezone.utc)
tomorrow = now + timedelta(days=1)

stamp = now.isoformat()
parsed = datetime.fromisoformat("2026-11-21T09:30:00Z")

local = now.astimezone(ZoneInfo("Asia/Dhaka"))
print(local.strftime("%Y-%m-%d %H:%M"))
```

</td>
</tr>
</table>

⚠️ A Swift `Date` is always an absolute instant. A Python `datetime` can be "naive" (no time zone). Always create aware ones with `timezone.utc`.

Date arithmetic is simpler than `Calendar`/`DateComponents`:

```python
from datetime import datetime, date, timedelta

tomorrow = date.today() + timedelta(days=1)            # dates without times
age = datetime(2026, 11, 21) - datetime(2026, 1, 1)    # subtracting gives a timedelta
print(age.days, age.total_seconds())                   # → 324 27993600.0
```

⚠️ `datetime.utcnow()` returns a *naive* datetime and is deprecated (3.12). Use `datetime.now(timezone.utc)`. Never mix naive and aware datetimes; comparisons raise `TypeError`.

## `re`

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
let text = "contact: ada@example.com"
let email = /(?<user>[\w.+-]+)@(?<domain>[\w-]+\.[\w.]+)/
if let m = text.firstMatch(of: email) {
    print(m.user, m.domain)
}
let digits = "a1 b22".matches(of: /\d+/).map { String($0.output) }
```

</td>
<td>

```python
import re

text = "contact: ada@example.com"
email = re.compile(r"(?P<user>[\w.+-]+)@(?P<domain>[\w-]+\.[\w.]+)")
if m := email.search(text):
    print(m["user"], m["domain"])      # → ada example.com

digits = re.findall(r"\d+", "a1 b22")  # → ['1', '22']
```

</td>
</tr>
</table>

Named groups are written `(?P<name>...)` in Python. Use raw strings (`r"..."`) so backslashes reach the regex engine untouched.

```python
import re

re.sub(r"[^\w\-. ]", "", "A: Light/in?").replace(" ", "_")   # sanitise filenames → 'A_Lightin'
re.split(r"[,;]\s*", "a, b;c")                   # → ['a', 'b', 'c']
re.fullmatch(r"[A-Z]{3}-\d{4}", "ABC-1234") is not None   # whole-string match → True
```

Always use raw strings (`r"..."`) for patterns. `re.search` scans; `re.match` anchors at the start; `re.fullmatch` must match everything.

## `argparse`

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
import ArgumentParser   // swift-argument-parser package

@main
struct Export: ParsableCommand {
    @Argument var source: String
    @Option(name: .shortAndLong) var output = "out.json"
    @Option var limit = 100
    @Flag var dryRun = false

    func run() { print(source, limit, dryRun) }
}
```

</td>
<td>

```python
import argparse

parser = argparse.ArgumentParser(description="Export orders")
parser.add_argument("source")
parser.add_argument("-o", "--output", default="out.json")
parser.add_argument("--limit", type=int, default=100)
parser.add_argument("--dry-run", action="store_true")

args = parser.parse_args(["orders.csv", "--limit", "5", "--dry-run"])
print(args.source, args.limit, args.dry_run)   # → orders.csv 5 True
```

</td>
</tr>
</table>

For the declarative, property-wrapper style of Swift Argument Parser, look at `typer`, which builds the CLI from a function's type hints.

For larger CLIs, `typer` (type-hint driven) or `click` are more ergonomic.

## `subprocess`

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
import Foundation

let p = Process()
p.executableURL = URL(fileURLWithPath: "/usr/bin/env")
p.arguments = ["git", "--version"]
let pipe = Pipe()
p.standardOutput = pipe
try p.run()
p.waitUntilExit()
let out = String(decoding: pipe.fileHandleForReading.readDataToEndOfFile(),
                 as: UTF8.self)
```

</td>
<td>

```python
import subprocess

result = subprocess.run(
    ["git", "--version"],
    capture_output=True, text=True, check=True, timeout=10,
)
print(result.stdout.strip())
```

</td>
</tr>
</table>

⚠️ Avoid `shell=True` and `os.system()` with any interpolated input — that's command injection. If a program needs to run commands chosen by a user or an LLM, allow-list them.

## `statistics` and `math`

```python
import statistics as st
import math

data = [12.5, 14.1, 13.8, 15.2, 11.9]
st.mean(data), st.median(data), st.stdev(data), st.quantiles(data, n=4)
slope, intercept = st.linear_regression([1, 2, 3], [2, 4, 6.1])
math.prod([1, 2, 3]), math.isqrt(17), math.dist((0, 0), (3, 4))   # → 6, 4, 5.0
```

## `heapq`, `bisect`, `secrets`, `uuid`

```python
import heapq, bisect, secrets, uuid

heapq.nlargest(2, [5, 1, 9, 3])                  # → [9, 5]
tasks: list[tuple[int, str]] = []
heapq.heappush(tasks, (2, "write")); heapq.heappush(tasks, (1, "read"))
heapq.heappop(tasks)                             # → (1, 'read')  min-heap

scores = [10, 20, 30]
bisect.insort(scores, 25)                        # keep sorted on insert

secrets.token_urlsafe(16)                        # tokens, API keys, password resets
secrets.choice("abcdef")                         # use secrets, not random, for security
uuid.uuid4().hex
```

⚠️ `random` is for simulations and sampling. For passwords, tokens or short codes that must be unguessable, use `secrets`.

## `sqlite3`

```python
import sqlite3
from contextlib import closing

with closing(sqlite3.connect(":memory:")) as conn:   # closing() actually closes the connection
    conn.row_factory = sqlite3.Row                     # rows behave like dicts
    with conn:                                         # transaction: commit or rollback
        conn.execute("CREATE TABLE prices (coin TEXT, price REAL, at TEXT)")
        conn.executemany(
            "INSERT INTO prices VALUES (?, ?, ?)",     # always parameterise — never f-strings
            [("btc", 64000.0, "2026-11-21"), ("eth", 3100.0, "2026-11-21")],
        )
    row = conn.execute("SELECT * FROM prices WHERE coin = ?", ("btc",)).fetchone()
    print(dict(row))
```

⚠️ `with sqlite3.connect(...) as conn:` manages the **transaction**, not the connection — it does not close it. Use `contextlib.closing` or call `conn.close()`.

---

[← 11. Testing and Code Quality](11-testing-and-quality.md) · [Contents](../README.md) · [13. Python for Data Engineering and AI →](13-data-engineering.md)
