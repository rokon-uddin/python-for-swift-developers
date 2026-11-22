# 09. File Handling and Data Serialization

[← 08. Functional Programming and Pythonic Patterns](08-functional-patterns.md) · [Contents](../README.md) · [10. Concurrency and Asynchronous Programming →](10-concurrency.md)

> **In Swift terms:** `pathlib.Path` replaces `URL(fileURLWithPath:)` + `FileManager`, `open()` + `with` replaces `FileHandle` + `defer`, and the `json` module returns plain dicts and lists — there's no `Codable`. For typed decoding, use dataclasses or Pydantic.

## `pathlib` first

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
import Foundation

let root = URL(fileURLWithPath: #filePath).deletingLastPathComponent()
let dataDir = root.appendingPathComponent("data")
try FileManager.default.createDirectory(
    at: dataDir, withIntermediateDirectories: true)

let report = dataDir.appendingPathComponent("report.txt")
try "hello\n".write(to: report, atomically: true, encoding: .utf8)
let text = try String(contentsOf: report, encoding: .utf8)
FileManager.default.fileExists(atPath: report.path)
```

</td>
<td>

```python
from pathlib import Path

root = Path(__file__).resolve().parent
data_dir = root / "data"                    # `/` joins paths
data_dir.mkdir(parents=True, exist_ok=True)

report = data_dir / "report.txt"
report.write_text("hello\n", encoding="utf-8")
text = report.read_text(encoding="utf-8")
report.exists()
```

</td>
</tr>
</table>

```python
from pathlib import Path

root = Path(__file__).resolve().parent
data_dir = root / "data"
data_dir.mkdir(parents=True, exist_ok=True)
report = data_dir / "report.txt"
report.write_text("hello\n", encoding="utf-8")

print(report.name, report.stem, report.suffix, report.parent.name)

for csv_file in sorted(data_dir.glob("*.csv")): ...      # non-recursive
for py in root.rglob("*.py"): ...                        # recursive
report.with_suffix(".md")                                # new Path, nothing renamed
report.exists(), report.is_file(), report.stat().st_size
report.unlink(missing_ok=True)
```

`shutil` covers what `pathlib` doesn't: `copy2`, `copytree`, `move`, `rmtree`, `disk_usage`, `which`, `make_archive`.

## Files and `open()`

```python
with open("orders.txt", "w", encoding="utf-8") as f:   # file is closed even if an exception occurs
    f.write("order 1001 - 4 items\n")

with open("orders.txt", "a", encoding="utf-8") as f:   # append
    print("order 1002 - 2 items", file=f)

with open("orders.txt", encoding="utf-8") as f:
    for line in f:                                     # streams line by line
        print(line.rstrip("\n"))
```

| Mode | Meaning |
|---|---|
| `r` / `w` / `a` / `x` | read / truncate+write / append / create (fail if exists) |
| `b` | binary (`rb`, `wb`) — returns `bytes` |
| `+` | read and write |

⚠️ Always pass `encoding="utf-8"` for text on Python < 3.15. Before 3.15 the default is platform-dependent (cp1252 on many Windows machines); 3.15 makes UTF-8 the default (PEP 686).

Atomic writes (no half-written file if the process dies):

```python
import os
import tempfile
from pathlib import Path

def atomic_write(path: Path, content: str) -> None:
    with tempfile.NamedTemporaryFile("w", dir=path.parent, delete=False, encoding="utf-8") as tmp:
        tmp.write(content)
    os.replace(tmp.name, path)       # atomic on POSIX and Windows
```

## Reading large files efficiently

```python
def iter_chunks(path: str, size: int = 1 << 20):
    with open(path, "rb") as f:
        while chunk := f.read(size):          # walrus + read loop
            yield chunk

import hashlib
def sha256(path: str) -> str:
    h = hashlib.sha256()
    for chunk in iter_chunks(path):
        h.update(chunk)
    return h.hexdigest()
```

Iterate lines rather than calling `.read()` / `.readlines()` on big files. For huge tabular files, use Polars `scan_csv` / PyArrow datasets or pandas `chunksize` (see [section 13](13-data-engineering.md)).

## CSV

```python
import csv
from pathlib import Path

path = Path("contacts.csv")
rows = [{"name": "Ada", "email": "ada@example.com"}, {"name": "Linus", "email": "l@example.com"}]

with path.open("w", newline="", encoding="utf-8") as f:     # newline="" is required by the csv module
    writer = csv.DictWriter(f, fieldnames=["name", "email"])
    writer.writeheader()
    writer.writerows(rows)

with path.open(newline="", encoding="utf-8") as f:
    for row in csv.DictReader(f):      # each row is dict[str, str] — everything is a string
        print(row["name"])
```

⚠️ Forgetting `newline=""` produces blank lines between rows on Windows. CSV values are always strings — convert types yourself (or use pandas/Polars).

## JSON

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
import Foundation

struct User: Codable {
    let id: Int
    let tags: [String]
}

let data = #"{"id": 1, "tags": ["a"]}"#.data(using: .utf8)!
let user = try JSONDecoder().decode(User.self, from: data)
let out = try JSONEncoder().encode(user)
```

</td>
<td>

```python
import json
from dataclasses import dataclass, asdict

@dataclass
class User:
    id: int
    tags: list[str]

raw = json.loads('{"id": 1, "tags": ["a"]}')   # → plain dict
user = User(**raw)                             # no type checking!
out = json.dumps(asdict(user))                 # '{"id": 1, "tags": ["a"]}'
print(user, out)
```

</td>
</tr>
</table>

`json.loads` gives you untyped dicts and lists; `User(**raw)` won't complain if `id` is a string. When the data comes from outside your program, decode with Pydantic (`User.model_validate_json(text)`), which validates like `Codable` does.

```python
import json
from datetime import datetime, date
from decimal import Decimal

payload = {"id": 1, "tags": ["a", "b"], "active": True, "score": None}
text = json.dumps(payload, indent=2, ensure_ascii=False)   # keep non-ASCII readable
data = json.loads(text)

with open("data.json", "w", encoding="utf-8") as f:
    json.dump(payload, f, indent=2)
with open("data.json", encoding="utf-8") as f:
    data = json.load(f)

def default(o: object):
    if isinstance(o, (datetime, date)):
        return o.isoformat()
    if isinstance(o, Decimal):
        return str(o)
    raise TypeError(f"not serialisable: {type(o).__name__}")

json.dumps({"at": datetime(2026, 1, 1)}, default=default)
```

JSON ↔ Python: object ↔ `dict`, array ↔ `list`, `null` ↔ `None`, `true` ↔ `True`. Tuples become arrays; dict keys become strings. For JSON Lines (`.jsonl`), write one `json.dumps(obj)` per line. For speed, `orjson` is a drop-in-ish faster alternative.

Flattening nested JSON (common before writing to CSV or a dataframe):

```python
def flatten(data: object, parent: str = "", sep: str = ".") -> dict[str, object]:
    items: dict[str, object] = {}
    if isinstance(data, dict):
        for k, v in data.items():
            items |= flatten(v, f"{parent}{sep}{k}" if parent else str(k), sep)
    elif isinstance(data, list):
        for i, v in enumerate(data):
            items |= flatten(v, f"{parent}{sep}{i}" if parent else str(i), sep)
    else:
        items[parent] = data
    return items

print(flatten({"user": {"id": 1, "roles": ["admin", "editor"]}}))
# → {'user.id': 1, 'user.roles.0': 'admin', 'user.roles.1': 'editor'}
```

## Other serialization formats

| Format | Module | Notes |
|---|---|---|
| TOML (read) | `tomllib` (3.11+) | Config files; open in binary mode: `tomllib.load(open(p, "rb"))` |
| YAML | `PyYAML` | Always `yaml.safe_load` |
| Pickle | `pickle` | Python-only, fast. ⚠️ Never unpickle untrusted data — it executes code |
| Parquet / Arrow | `pyarrow`, `polars` | Columnar, typed, compressed — the default for analytics data |
| SQLite | `sqlite3` | Built-in embedded database |
| Base64 | `base64` | Encoding, **not** encryption |

## Environment variables and secrets

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
import Foundation

let env = ProcessInfo.processInfo.environment
let apiKey = env["OPENAI_API_KEY"]          // String?
let port = Int(env["PORT"] ?? "8000") ?? 8000
```

</td>
<td>

```python
import os

api_key = os.environ.get("OPENAI_API_KEY")  # str | None
port = int(os.getenv("PORT", "8000"))
# os.environ["OPENAI_API_KEY"] raises KeyError when unset
```

</td>
</tr>
</table>

Environment values are always strings: parse booleans explicitly (`os.getenv("DEBUG", "false").lower() in {"1", "true", "yes"}`), because `bool("false")` is `True`.

For local development, keep secrets in a `.env` file loaded by `python-dotenv` (and `.env` in `.gitignore`, with a committed `.env.example`). For typed, validated configuration, use `pydantic-settings` (shown in [section 14](14-llm-applications.md)).

⚠️ Never hard-code API keys or database passwords in source. If one has been committed, rotate it — removing it from the latest commit doesn't remove it from history.

## Directory operations

```python
import shutil
from pathlib import Path

def organise_by_extension(folder: Path) -> None:
    buckets = {".pdf": "PDFs", ".jpg": "Images", ".jpeg": "Images", ".png": "Images", ".txt": "Text"}
    for entry in folder.iterdir():
        if entry.is_file():
            target = folder / buckets.get(entry.suffix.lower(), "Other")
            target.mkdir(exist_ok=True)
            shutil.move(entry, target / entry.name)
```

For event-driven folder watching, use the `watchdog` package (block with `observer.join()` or `time.sleep` — never a `while True: pass` busy loop).

---

[← 08. Functional Programming and Pythonic Patterns](08-functional-patterns.md) · [Contents](../README.md) · [10. Concurrency and Asynchronous Programming →](10-concurrency.md)
