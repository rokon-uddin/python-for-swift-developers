# 13. Python for Data Engineering and AI

[← 12. Python Standard Library](12-standard-library.md) · [Contents](../README.md) · [14. Python for LLM and AI Application Development →](14-llm-applications.md)

> **In Swift terms:** Swift is fast because it compiles to machine code, so a plain `for` loop over a million numbers is fine. A Python loop over a million numbers is roughly 50–100× slower. Data libraries such as NumPy, pandas and Polars solve this by running the loop in compiled code: you describe an operation on a whole column or array ("vectorisation") and the library does the iterating. Most of this section is about writing code that way.

## The landscape

| Library | Role | Mental model |
|---|---|---|
| **NumPy** | N-dimensional typed arrays, vectorised math | The foundation; everything else interoperates with it |
| **pandas** | DataFrames with an index; huge ecosystem | Eager, single-threaded (mostly), flexible |
| **Polars** | DataFrames on Apache Arrow, written in Rust | Multi-threaded, lazy query optimiser, strict types |
| **PyArrow** | Arrow columnar memory format, Parquet/IPC I/O | The interchange layer between tools |
| **PySpark** | Distributed DataFrames on a cluster | For data that doesn't fit on one machine |
| **SQLAlchemy** | SQL toolkit + ORM | Database access with connection pooling |
| **Pydantic** | Runtime validation and parsing | Schema at every system boundary |
| **Requests / httpx** | HTTP clients | `httpx` adds async and HTTP/2 |

## NumPy and vectorisation

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
let salaries: [Double] = loadSalaries()

// A plain loop is fast in Swift
var adjusted: [Double] = []
for s in salaries where s > 50_000 {
    adjusted.append(s * 1.1)
}
let mean = salaries.reduce(0, +) / Double(salaries.count)
```

</td>
<td>

```python
import numpy as np

salaries = np.array([42_000.0, 61_000.0, 75_500.0, 38_000.0])

# Don't loop in Python: operate on the whole array at once
adjusted = salaries[salaries > 50_000] * 1.1   # filter + map, in C
mean = salaries.mean()
print(adjusted, round(mean))                   # → [67100. 83050.] 54125
```

</td>
</tr>
</table>

`salaries > 50_000` produces an array of booleans, and indexing with it keeps only the `True` positions. The same filter-then-map style carries over to pandas and Polars columns.

The rule: **no Python loops over numeric data**. Express operations on whole arrays and let compiled code do the iteration.

```python
import numpy as np

rng = np.random.default_rng(42)                       # modern RNG API (not np.random.seed)
years = rng.uniform(0.5, 10, size=1_000).round(2)
salary = 30_000 + years * 6_000 + rng.normal(0, 4_000, size=1_000)

print(years.dtype, years.shape, salary.mean().round(0))
high = salary[salary > 80_000]                        # boolean mask indexing
z = (salary - salary.mean()) / salary.std()           # broadcasting a scalar over the array
matrix = np.arange(12).reshape(3, 4)
print(matrix.sum(axis=0), matrix @ matrix.T)          # column sums, matrix multiply
```

⚠️ Slices of NumPy arrays are **views** (unlike Python lists). Mutating a slice mutates the original — call `.copy()` when you need independence. Choosing `float32`/`int32` instead of the 64-bit defaults halves memory.

## pandas essentials

```python
import pandas as pd

df = pd.DataFrame({
    "order_id": [1, 2, 3, 4, 5],
    "customer": ["ada", "bo", "ada", "cy", None],
    "amount": [120.0, None, 80.5, 42.0, 99.0],
    "created": ["2026-01-03", "2026-01-04", "2026-02-01", "2026-02-03", "2026-02-10"],
})

clean = (
    df
    .dropna(subset=["customer"])
    .assign(
        amount=lambda d: d["amount"].fillna(0.0),
        created=lambda d: pd.to_datetime(d["created"]),
        month=lambda d: pd.to_datetime(d["created"]).dt.to_period("M").astype(str),
    )
)

summary = (
    clean.groupby(["customer", "month"], as_index=False)
    .agg(orders=("order_id", "count"), revenue=("amount", "sum"))
    .sort_values("revenue", ascending=False)
)
print(summary)

big = clean.loc[clean["amount"] > 50, ["order_id", "amount"]]   # label-based selection
first_row = clean.iloc[0]                                          # position-based
```

Things to know:

- Use `.loc[row_mask, columns]` for assignment. Chained indexing (`df[df.x > 1]["y"] = 0`) may write to a temporary copy; pandas 3.0 makes Copy-on-Write the default, so chained assignment never modifies the original.
- `df.apply(func, axis=1)` is a Python loop in disguise. Prefer vectorised column operations, `np.where`, `Series.map`, or `.str` / `.dt` accessors.
- Set explicit dtypes on load (`dtype={"id": "int64", "code": "category"}`); `category` dtype dramatically cuts memory for repeated strings.
- `df.info(memory_usage="deep")` shows the real memory footprint.
- Joins: `left.merge(right, on="id", how="left", validate="many_to_one")` — `validate` catches accidental row explosions.

Reading big files in chunks:

```python
# for chunk in pd.read_csv("events.csv", chunksize=100_000, usecols=["user", "amount"]):
#     process(chunk)
```

## Polars

```python
import polars as pl

orders = pl.DataFrame({
    "customer": ["ada", "bo", "ada", "cy"],
    "amount": [120.0, None, 80.5, 42.0],
    "created": ["2026-01-03", "2026-01-04", "2026-02-01", "2026-02-03"],
})

result = (
    orders.lazy()                                         # build a query plan
    .with_columns(
        pl.col("amount").fill_null(0.0),
        pl.col("created").str.to_date().dt.month().alias("month"),
    )
    .filter(pl.col("amount") > 0)
    .group_by("customer", "month")
    .agg(pl.len().alias("orders"), pl.col("amount").sum().alias("revenue"))
    .sort("revenue", descending=True)
    .collect()                                            # optimise + execute
)
print(result)

# Out-of-core: scan instead of read, so only needed columns/rows are loaded
# pl.scan_parquet("events/*.parquet").filter(pl.col("country") == "BD").select("user_id").collect()
```

Why I reach for Polars on new pipelines: expression API (no index, no `SettingWithCopy` ambiguity), automatic multi-threading, predicate/projection pushdown when scanning Parquet, and streaming execution for larger-than-memory data. pandas remains the lingua franca for plotting, scikit-learn and many libraries — convert with `df.to_pandas()` / `pl.from_pandas()` (zero-copy via Arrow where possible).

## PyArrow and Parquet

```python
import pyarrow as pa
import pyarrow.parquet as pq
import pyarrow.compute as pc

table = pa.table({"id": [1, 2, 3], "score": [0.5, 0.9, 0.7]})
pq.write_table(table, "scores.parquet", compression="zstd")

loaded = pq.read_table("scores.parquet", columns=["score"])   # column pruning
print(pc.mean(loaded["score"]))
```

| Format | Typed | Compressed | Columnar | Append-friendly | Use for |
|---|---|---|---|---|---|
| CSV | No | No | No | Yes | Interchange with humans/spreadsheets |
| JSON / JSONL | Loosely | No | No | JSONL yes | APIs, logs, nested records |
| Parquet | Yes | Yes | Yes | Partitioned datasets | Analytics storage, data lakes |
| Arrow IPC / Feather | Yes | Optional | Yes | No | Fast local interchange between processes/tools |

## PySpark (when one machine isn't enough)

```python
# from pyspark.sql import SparkSession, functions as F
# spark = SparkSession.builder.appName("orders").getOrCreate()
# df = spark.read.parquet("s3://bucket/orders/")
# daily = (df.filter(F.col("status") == "paid")
#            .groupBy(F.to_date("created_at").alias("day"))
#            .agg(F.sum("amount").alias("revenue")))
# daily.write.mode("overwrite").partitionBy("day").parquet("s3://bucket/daily/")
```

Spark is lazy like Polars: transformations build a plan; actions (`count`, `collect`, `write`) run it. Avoid Python UDFs where a built-in `F.` function exists — UDFs serialise every row to a Python worker.

## SQLAlchemy 2.0

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
import Foundation
import SwiftData

@Model
final class Order {
    var customer: String
    var amount: Double
    init(customer: String, amount: Double) {
        self.customer = customer
        self.amount = amount
    }
}

let descriptor = FetchDescriptor<Order>(
    predicate: #Predicate { $0.amount > 50 },
    sortBy: [SortDescriptor(\.amount, order: .reverse)])
let big = try context.fetch(descriptor)
```

</td>
<td>

```python
from sqlalchemy import create_engine, select
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, Session

class Base(DeclarativeBase): ...

class Order(Base):
    __tablename__ = "orders"
    id: Mapped[int] = mapped_column(primary_key=True)
    customer: Mapped[str]
    amount: Mapped[float]

engine = create_engine("sqlite:///:memory:")
Base.metadata.create_all(engine)
with Session(engine) as session:
    session.add(Order(customer="ada", amount=120)); session.commit()
    stmt = select(Order).where(Order.amount > 50).order_by(Order.amount.desc())
    print([o.customer for o in session.scalars(stmt)])   # → ['ada']
```

</td>
</tr>
</table>

The mental model is close to SwiftData/Core Data: a model class, a session/context, and query objects. `Order.amount > 50` builds a SQL expression rather than evaluating to a Bool, which plays the role of `#Predicate`.

Aggregations are written as query expressions, much like building a `FetchDescriptor`, but compiled to SQL `GROUP BY`:

```python
from sqlalchemy import func, select

# using the Order model above:
# stmt = (select(Order.customer, func.sum(Order.amount).label("total"))
#         .group_by(Order.customer)
#         .order_by(func.sum(Order.amount).desc()))
# for customer, total in session.execute(stmt):
#     print(customer, total)
```

`with Session(engine) as session, session.begin():` commits on success and rolls back on an exception, like a `ModelContext` save wrapped in `do`/`catch`.

For analytics, `pd.read_sql(stmt, engine)` / `pl.read_database(query, connection)` pull query results straight into a DataFrame. Use Alembic for schema migrations; `create_async_engine` + `AsyncSession` for asyncio services.

## Pydantic v2 — validation at the boundary

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
import Foundation

struct Address: Codable {
    let city: String
    let postalCode: String
    enum CodingKeys: String, CodingKey {
        case city, postalCode = "postal_code"
    }
}

struct User: Codable {
    let id: Int
    let name: String
    let age: Int?
    let address: Address?
}

let json = #"{"id": 1, "name": "Ada", "address": {"city": "Dhaka", "postal_code": "1212"}}"#
let user = try JSONDecoder().decode(User.self, from: Data(json.utf8))
let back = try JSONEncoder().encode(user)
```

</td>
<td>

```python
from pydantic import BaseModel, Field

class Address(BaseModel):
    city: str
    postal_code: str          # snake_case keys match JSON directly

class User(BaseModel):
    id: int
    name: str = Field(min_length=2)       # validation rules built in
    age: int | None = None
    address: Address | None = None

json_text = '{"id": 1, "name": "Ada", "address": {"city": "Dhaka", "postal_code": "1212"}}'
user = User.model_validate_json(json_text)     # ≈ JSONDecoder().decode
back = user.model_dump_json()                  # ≈ JSONEncoder().encode
print(user.address.city if user.address else None)   # → Dhaka
```

</td>
</tr>
</table>

Pydantic is the closest thing Python has to `Codable`, and it goes further: it validates constraints (lengths, ranges, patterns) and, by default, converts compatible values (`"1"` → `1`). It raises one `ValidationError` listing every problem instead of stopping at the first.

Pydantic parses and validates data into typed models, with lax type coercion by default (`"42"` → `42`) and a strict mode when you don't want that.

```python
from datetime import datetime
from typing import Annotated, Literal
from pydantic import BaseModel, Field, EmailStr, ValidationError, field_validator, model_validator, computed_field, ConfigDict

class Address(BaseModel):
    street: str
    city: str
    postal_code: Annotated[str, Field(pattern=r"^\d{4,6}$")]

class User(BaseModel):
    model_config = ConfigDict(str_strip_whitespace=True, extra="forbid")

    id: int
    name: Annotated[str, Field(min_length=3, max_length=50)]
    email: str
    age: Annotated[int, Field(ge=0, le=150)] = 0
    plan: Literal["free", "pro"] = "free"
    tags: list[str] = []                     # safe in Pydantic: defaults are copied per instance
    address: Address | None = None           # nested model
    created_at: datetime

    @field_validator("email")
    @classmethod
    def normalise_email(cls, v: str) -> str:
        return v.lower()

    @computed_field
    @property
    def is_adult(self) -> bool:
        return self.age >= 18

class DateRange(BaseModel):
    start: datetime
    end: datetime

    @model_validator(mode="after")           # cross-field validation, runs on the built instance
    def check_order(self) -> "DateRange":
        if self.start >= self.end:
            raise ValueError("end must be after start")
        return self

user = User.model_validate({
    "id": "101", "name": "  Ada  ", "email": "ADA@Example.com",
    "age": 36, "created_at": "2026-11-21T09:30:00Z",
    "address": {"street": "1 Main", "city": "Dhaka", "postal_code": "1212"},
})
print(user.id, user.name, user.email, user.is_adult)      # → 101 Ada ada@example.com True
print(user.model_dump(exclude_none=True))                  # → dict (computed fields included)
print(user.model_dump_json(include={"id", "email"}))       # → '{"id":101,"email":"ada@example.com"}'

try:
    User.model_validate({"id": "abc", "name": "Al", "email": "x", "created_at": "now"})
except ValidationError as e:
    print(e.error_count(), [err["loc"] for err in e.errors()])
```

Notes from using it in anger:

- **v1 → v2 renames:** `Field(regex=...)` → `pattern=`, `@validator` → `@field_validator`, `@root_validator` → `@model_validator`, `.dict()`/`.json()` → `.model_dump()`/`.model_dump_json()`, `parse_obj` → `model_validate`, `json_encoders` → `@field_serializer`.
- `mode="before"` validators receive raw input (use them to clean `"$4.44"` into `4.44`); `mode="after"` receives the typed value. A field validator must **return** the value.
- Self-referencing models (`replies: list["Comment"] = []`) work; call `Model.model_rebuild()` if a forward reference is defined later.
- Define leaf models first, keep models flat where possible, and don't overuse `computed_field` — it is recomputed on every access and every dump.
- `TypeAdapter(list[User]).validate_python(rows)` validates types that aren't models.
- `EmailStr` needs `pip install "pydantic[email]"`.

```python
from pydantic import BaseModel, field_serializer
from datetime import datetime

class Event(BaseModel):
    at: datetime

    @field_serializer("at")
    def fmt(self, at: datetime) -> str:
        return at.strftime("%d-%m-%Y %H:%M")

print(Event(at=datetime(2026, 3, 15, 14, 30)).model_dump_json())   # → {"at":"15-03-2026 14:30"}
```

## HTTP clients and REST APIs

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
import Foundation

func getJSON(_ url: URL) async throws -> [String: Any] {
    var req = URLRequest(url: url, timeoutInterval: 10)
    req.setValue("application/json", forHTTPHeaderField: "Accept")
    let (data, resp) = try await URLSession.shared.data(for: req)
    guard let http = resp as? HTTPURLResponse,
          (200..<300).contains(http.statusCode) else {
        throw URLError(.badServerResponse)
    }
    return try JSONSerialization.jsonObject(with: data) as! [String: Any]
}
```

</td>
<td>

```python
import httpx

def get_json(url: str) -> dict:
    with httpx.Client(timeout=10) as client:
        r = client.get(url, headers={"Accept": "application/json"})
        r.raise_for_status()          # non-2xx → httpx.HTTPStatusError
        return r.json()
```

</td>
</tr>
</table>

Unlike `URLSession`, `requests` and `httpx` have no async-by-default model; `httpx.AsyncClient` gives you the `async` version ([chapter 10](10-concurrency.md#async-http)).

⚠️ `requests` has **no default timeout** — a call can hang forever. Always pass `timeout=`. Reuse a `Session`/`Client` across calls for connection pooling. Don't build query strings by hand; pass `params=` (and use `urllib.parse.urljoin` for relative links when scraping).

## Data pipeline patterns

A small, testable shape I use for batch jobs — **extract → validate → transform → load**, each step a pure-ish function:

```python
from collections.abc import Iterable, Iterator
from pydantic import BaseModel, ValidationError
import csv, io, logging

log = logging.getLogger("pipeline")

class Row(BaseModel):
    date: str
    city: str
    temperature: float
    condition: str

def extract(text: str) -> Iterator[dict[str, str]]:
    yield from csv.DictReader(io.StringIO(text))

def validate(rows: Iterable[dict[str, str]]) -> Iterator[Row]:
    for i, raw in enumerate(rows, start=1):
        try:
            yield Row.model_validate(raw)
        except ValidationError as exc:
            log.warning("row %d rejected: %s", i, exc.errors()[0]["msg"])   # dead-letter instead of crashing

def transform(rows: Iterable[Row]) -> dict[str, float]:
    totals: dict[str, list[float]] = {}
    for r in rows:
        totals.setdefault(r.city, []).append(r.temperature)
    return {city: round(sum(v) / len(v), 1) for city, v in totals.items()}

raw = "date,city,temperature,condition\n2026-06-11,Delhi,36.5,Clear\n2026-06-12,Delhi,oops,Sunny\n2026-06-12,Dhaka,33.0,Rain\n"
print(transform(validate(extract(raw))))   # → {'Delhi': 36.5, 'Dhaka': 33.0}
```

Principles:

- **Idempotent loads:** re-running a job for the same date must not duplicate data (upserts, `overwrite` partitions, or a unique key).
- **Validate at the edges**, use plain typed objects inside.
- **Stream** (generators, chunked reads, lazy frames) rather than loading everything.
- **Push computation down** to the database or the columnar engine instead of pulling rows into Python.
- **Log counts** at every stage (read, rejected, written) — the cheapest data-quality monitoring there is.

## Memory-efficient processing — checklist

1. Iterate, don't materialise (`for line in f`, generator expressions, `itertools`).
2. Read only needed columns (`usecols=`, `columns=`, `select`).
3. Use compact dtypes (`int32`, `float32`, `category`, Arrow strings).
4. Use Parquet with column pruning and partition filters.
5. Use lazy engines (Polars `scan_*`, DuckDB, Spark) for larger-than-memory data.
6. Measure: `sys.getsizeof` (shallow only), `tracemalloc`, `df.memory_usage(deep=True)`.

---

[← 12. Python Standard Library](12-standard-library.md) · [Contents](../README.md) · [14. Python for LLM and AI Application Development →](14-llm-applications.md)
