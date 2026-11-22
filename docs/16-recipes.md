# 16. Practical Code Recipes

[← 15. Common Pitfalls and Best Practices](15-pitfalls.md) · [Contents](../README.md)

## Filtering and transforming lists

```python
orders = [{"id": 1, "total": 120, "status": "paid"}, {"id": 2, "total": 40, "status": "refunded"}]
paid_totals = [o["total"] for o in orders if o["status"] == "paid"]
ids_by_status = {o["status"]: o["id"] for o in orders}
flattened = [x for row in [[1, 2], [3]] for x in row]
first_big = next((o for o in orders if o["total"] > 100), None)    # first match or None
```

## Merging dictionaries

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
let defaults = ["timeout": 10, "retries": 3]
let overrides = ["retries": 5]
let config = defaults.merging(overrides) { _, new in new }
```

</td>
<td>

```python
defaults = {"timeout": 10, "retries": 3}
overrides = {"retries": 5}
config = defaults | overrides        # right side wins
print(config)                        # → {'timeout': 10, 'retries': 5}
```

</td>
</tr>
</table>

Merging nested dictionaries (neither language has this built in):

```python
def deep_merge(a: dict, b: dict) -> dict:
    out = dict(a)
    for k, v in b.items():
        out[k] = deep_merge(out[k], v) if isinstance(v, dict) and isinstance(out.get(k), dict) else v
    return out

print(deep_merge({"db": {"host": "x", "port": 1}}, {"db": {"port": 2}}))   # → {'db': {'host': 'x', 'port': 2}}
```

## Sorting complex objects

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
struct Book { let title: String; let rating: Double; let year: Int }
let books = [Book(title: "B", rating: 4.5, year: 2020),
             Book(title: "A", rating: 4.5, year: 2018)]

let sorted = books.sorted {
    ($0.rating, $1.title) > ($1.rating, $0.title)  // rating desc, title asc
}
let newest = books.max { $0.year < $1.year }
```

</td>
<td>

```python
from dataclasses import dataclass

@dataclass
class Book:
    title: str
    rating: float
    year: int

books = [Book("B", 4.5, 2020), Book("A", 4.5, 2018)]
ordered = sorted(books, key=lambda b: (-b.rating, b.title))   # rating desc, title asc
newest = max(books, key=lambda b: b.year)
print([b.title for b in ordered], newest.title)   # → ['A', 'B'] B
```

</td>
</tr>
</table>

## Removing duplicates

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
let items = ["b", "a", "b", "c", "a"]
let unique = Set(items)                  // order lost
var seen = Set<String>()
let ordered = items.filter { seen.insert($0).inserted }   // keeps order
```

</td>
<td>

```python
items = ["b", "a", "b", "c", "a"]
unique = set(items)                      # order lost
ordered = list(dict.fromkeys(items))     # keeps first occurrence
print(ordered)                           # → ['b', 'a', 'c']
```

</td>
</tr>
</table>

Deduplicating by a key (Swift: a `Set` of seen keys inside `filter`):

```python
def unique_by(items, key):
    seen = set()
    for item in items:
        k = key(item)
        if k not in seen:
            seen.add(k)
            yield item

people = [{"email": "A@x.com"}, {"email": "a@x.com"}, {"email": "b@x.com"}]
print(len(list(unique_by(people, key=lambda p: p["email"].lower()))))   # → 2
```

## Reading large files

```python
from collections import Counter
from collections.abc import Iterator

def tail_errors(path: str) -> Iterator[str]:
    with open(path, encoding="utf-8", errors="replace") as f:   # don't crash on bad bytes
        for line in f:
            if " ERROR " in line:
                yield line.rstrip()

def count_levels(path: str) -> Counter[str]:
    counts: Counter[str] = Counter()
    with open(path, encoding="utf-8") as f:
        for line in f:
            parts = line.split(maxsplit=3)
            if len(parts) >= 3:
                counts[parts[2]] += 1
    return counts
```

## Processing CSV and JSON

```python
import csv
import json
from pathlib import Path

def csv_to_json(src: Path, dst: Path) -> int:
    with src.open(newline="", encoding="utf-8") as f:
        rows = list(csv.DictReader(f))
    dst.write_text(json.dumps(rows, indent=2, ensure_ascii=False), encoding="utf-8")
    return len(rows)

def json_to_csv(src: Path, dst: Path) -> int:
    records: list[dict] = json.loads(src.read_text(encoding="utf-8"))
    fieldnames = sorted({k for r in records for k in r})      # union of keys, not just the first row's
    with dst.open("w", newline="", encoding="utf-8") as f:
        writer = csv.DictWriter(f, fieldnames=fieldnames, extrasaction="ignore")
        writer.writeheader()
        writer.writerows(records)                              # missing keys → empty cells
    return len(records)
```

## Making REST API calls and handling HTTP errors

```python
import logging
import httpx

log = logging.getLogger(__name__)

class ApiError(Exception):
    def __init__(self, status: int, body: str) -> None:
        super().__init__(f"HTTP {status}: {body[:200]}")
        self.status = status

def fetch_markets(client: httpx.Client, per_page: int = 10) -> list[dict]:
    try:
        r = client.get(
            "https://api.coingecko.com/api/v3/coins/markets",
            params={"vs_currency": "usd", "order": "market_cap_desc", "per_page": per_page, "page": 1},
        )
        r.raise_for_status()
    except httpx.TimeoutException:
        log.warning("timed out")
        raise
    except httpx.HTTPStatusError as e:
        raise ApiError(e.response.status_code, e.response.text) from e
    except httpx.RequestError as e:                     # DNS, connection refused, TLS…
        raise ApiError(0, str(e)) from e
    return r.json()

# with httpx.Client(timeout=10, headers={"User-Agent": "my-tool/1.0"}) as client:
#     markets = fetch_markets(client)
```

## Working with dates

```python
from datetime import datetime, timedelta, timezone, date
from zoneinfo import ZoneInfo

now_utc = datetime.now(timezone.utc)
local = now_utc.astimezone(ZoneInfo("Asia/Dhaka"))
start_of_day = local.replace(hour=0, minute=0, second=0, microsecond=0)
last_7_days = [date.today() - timedelta(days=i) for i in range(7)]
parsed = datetime.strptime("21/11/2026 14:30", "%d/%m/%Y %H:%M").replace(tzinfo=ZoneInfo("Asia/Dhaka"))
iso = parsed.isoformat()
minutes_old = round((now_utc - parsed).total_seconds() / 60)
mins, secs = divmod(125, 60); print(f"{mins:02}:{secs:02}")     # → 02:05
```

## Logging setup

```python
import logging
import sys

def setup_logging(level: str = "INFO") -> None:
    handler = logging.StreamHandler(sys.stdout)
    handler.setFormatter(logging.Formatter("%(asctime)s | %(levelname)-7s | %(name)s | %(message)s"))
    root = logging.getLogger()
    root.handlers[:] = [handler]
    root.setLevel(level)
    logging.getLogger("httpx").setLevel(logging.WARNING)     # quiet noisy libraries

setup_logging()
logging.getLogger("app").info("started")
```

## Writing decorators

```python
import functools
import time

def retry(times: int = 3, delay: float = 0.1, exceptions: tuple[type[Exception], ...] = (Exception,)):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(1, times + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions:
                    if attempt == times:
                        raise
                    time.sleep(delay * attempt)
        return wrapper
    return decorator

calls = {"n": 0}

@retry(times=3, exceptions=(ConnectionError,))
def flaky() -> str:
    calls["n"] += 1
    if calls["n"] < 3:
        raise ConnectionError("try again")
    return "ok"

print(flaky(), calls["n"])     # → ok 3
```

## Creating generators

```python
from collections.abc import Iterator
from itertools import batched           # 3.12+

def paginate(fetch_page, start: int = 1) -> Iterator[dict]:
    """Yield items across pages until an empty page is returned."""
    page = start
    while items := fetch_page(page):
        yield from items
        page += 1

pages = {1: [{"id": 1}, {"id": 2}], 2: [{"id": 3}]}
print([x["id"] for x in paginate(lambda p: pages.get(p, []))])   # → [1, 2, 3]
print(list(batched(range(5), 2)))                                  # → [(0, 1), (2, 3), (4,)]
```

## Executing concurrent tasks (threads)

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import time

def download(url: str) -> tuple[str, int]:
    time.sleep(0.05)                    # requests.get(url, timeout=10)
    return url, 1024

urls = [f"https://example.com/file/{i}" for i in range(20)]
results: dict[str, int] = {}
with ThreadPoolExecutor(max_workers=8) as pool:
    futures = [pool.submit(download, u) for u in urls]
    for fut in as_completed(futures):
        try:
            url, size = fut.result()
            results[url] = size
        except Exception as exc:        # one failure doesn't kill the batch
            print("failed:", exc)
print(len(results))                     # → 20
```

## Async API calls

```python
import asyncio
import httpx

async def fetch_json(client: httpx.AsyncClient, url: str, sem: asyncio.Semaphore) -> dict | None:
    async with sem:
        try:
            r = await client.get(url)
            r.raise_for_status()
            return r.json()
        except httpx.HTTPError as exc:
            print(f"{url}: {exc!r}")
            return None

async def fetch_many(urls: list[str], limit: int = 10) -> list[dict | None]:
    sem = asyncio.Semaphore(limit)
    async with httpx.AsyncClient(timeout=10) as client:
        return await asyncio.gather(*(fetch_json(client, u, sem) for u in urls))

# results = asyncio.run(fetch_many([f"https://httpbin.org/json?i={i}" for i in range(20)]))
```

## Creating dataclasses

```python
from dataclasses import dataclass, field, asdict, replace
from datetime import datetime, timezone

@dataclass(frozen=True, slots=True, kw_only=True)
class Note:
    title: str
    body: str
    tags: tuple[str, ...] = ()
    created: datetime = field(default_factory=lambda: datetime.now(timezone.utc))

note = Note(title="Idea", body="Use Polars", tags=("data",))
edited = replace(note, body="Use Polars lazily")
print(asdict(edited)["body"])
```

## Validating input

```python
from pydantic import BaseModel, Field, ValidationError

class SignupForm(BaseModel):
    username: str = Field(min_length=4, max_length=30, pattern=r"^[a-z0-9_]+$")
    password: str = Field(min_length=8)
    age: int = Field(ge=13)

def parse_form(data: dict) -> SignupForm | list[str]:
    try:
        return SignupForm.model_validate(data)
    except ValidationError as e:
        return [f"{'.'.join(map(str, err['loc']))}: {err['msg']}" for err in e.errors()]

print(parse_form({"username": "Ad!", "password": "short", "age": "12"}))

def prompt_float(prompt: str) -> float:          # interactive CLI input loop
    while True:
        try:
            return float(input(prompt))
        except ValueError:
            print("Please enter a number.")
```

## Reading environment variables

```python
import os

def env_bool(name: str, default: bool = False) -> bool:
    raw = os.getenv(name)
    return default if raw is None else raw.strip().lower() in {"1", "true", "yes", "on"}

def env_required(name: str) -> str:
    try:
        return os.environ[name]
    except KeyError:
        raise SystemExit(f"Missing required environment variable: {name}") from None

DEBUG = env_bool("DEBUG")
WORKERS = int(os.getenv("WORKERS", "4"))
```

## Writing CLI scripts

```python
#!/usr/bin/env python3
"""Count words in one or more text files."""
from __future__ import annotations

import argparse
import logging
import sys
from collections import Counter
from pathlib import Path

log = logging.getLogger("wordcount")

def parse_args(argv: list[str] | None = None) -> argparse.Namespace:
    p = argparse.ArgumentParser(description=__doc__)
    p.add_argument("files", nargs="+", type=Path)
    p.add_argument("-n", "--top", type=int, default=10)
    p.add_argument("-v", "--verbose", action="store_true")
    return p.parse_args(argv)

def main(argv: list[str] | None = None) -> int:
    args = parse_args(argv)
    logging.basicConfig(level=logging.DEBUG if args.verbose else logging.INFO)
    counts: Counter[str] = Counter()
    for path in args.files:
        if not path.is_file():
            log.error("not a file: %s", path)
            return 2
        with path.open(encoding="utf-8") as f:
            for line in f:
                counts.update(w.strip(".,!?;:").lower() for w in line.split())
    for word, n in counts.most_common(args.top):
        print(f"{n:>6}  {word}")
    return 0

if __name__ == "__main__":
    sys.exit(main())
```

## Simple key–value cache with TTL

```python
import time
from typing import Any

class TTLCache:
    def __init__(self, ttl: float) -> None:
        self.ttl = ttl
        self._data: dict[str, tuple[float, Any]] = {}

    def get(self, key: str) -> Any | None:
        item = self._data.get(key)
        if item is None:
            return None
        expires, value = item
        if time.monotonic() > expires:            # monotonic: immune to wall-clock changes
            del self._data[key]
            return None
        return value

    def set(self, key: str, value: Any) -> None:
        self._data[key] = (time.monotonic() + self.ttl, value)
```

## Encrypting small secrets at rest

```python
from cryptography.fernet import Fernet       # pip install cryptography

key = Fernet.generate_key()                   # store in a secret manager / env var, NOT next to the data
f = Fernet(key)
token = f.encrypt(b"api-password")
print(f.decrypt(token))                       # → b'api-password'
```

Base64 is not encryption; it's reversible encoding. Fernet (AES-128-CBC + HMAC) is authenticated symmetric encryption.

---

[← 15. Common Pitfalls and Best Practices](15-pitfalls.md) · [Contents](../README.md)
