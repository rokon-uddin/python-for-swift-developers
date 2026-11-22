# 10. Concurrency and Asynchronous Programming

[← 09. File Handling and Data Serialization](09-files-and-serialization.md) · [Contents](../README.md) · [11. Testing and Code Quality →](11-testing-and-quality.md)

> **In Swift terms:** `async`/`await` will look familiar, but the runtime underneath is very different. Swift runs async code on a pool of threads across all cores and the compiler checks for data races. Python's `asyncio` runs all async code on **one thread**, and nothing checks for races. On top of that, standard Python has the GIL, so threads can't run Python code on several cores at once. The rest of this section explains what to use instead.

**Two ideas to hold on to before reading further:**

1. **asyncio = cooperative multitasking on one thread.** While one coroutine is waiting at an `await` (for a network reply, a timer), the event loop runs another. If a coroutine *doesn't* await — a CPU loop or a blocking call like `time.sleep` — every other coroutine stops. In Swift terms: imagine all your async functions were `@MainActor` and you weren't allowed to block the main thread.
2. **GIL = only one thread executes Python code at a time.** Threads still help when they spend most of their time *waiting* (network, disk), because waiting releases the lock. They don't help for number crunching; for that you use separate processes, each with its own interpreter.

The full mapping table is at [Swift Concurrency ↔ asyncio](#swift-concurrency--asyncio).

## The decision table

| Workload | Tool | Why |
|---|---|---|
| Many network calls, async-capable libraries | **`asyncio`** | Thousands of concurrent I/O operations on one thread, low overhead |
| Blocking I/O with sync libraries (`requests`, DB drivers, file I/O) | **Threads** (`ThreadPoolExecutor`) | The GIL is released while waiting on I/O |
| CPU-bound pure Python (parsing, math loops, compression) | **Processes** (`ProcessPoolExecutor`) | Each process has its own interpreter and GIL |
| CPU-bound numeric work | **NumPy / Polars / PyArrow** | Vectorised C/Rust code releases the GIL and is 10–100× faster than any parallel Python loop |
| Background jobs that outlive a request | **A task queue** (RQ, Celery, Arq, Dramatiq) | Durable, retryable, horizontally scalable workers |

**Concurrency** is about dealing with many things at once (interleaving while waiting). **Parallelism** is doing many things at the same instant (multiple cores). asyncio and threads give you concurrency; processes give you parallelism.

## The Global Interpreter Lock (GIL)

In the default CPython build, a single mutex — the GIL — means only one thread executes Python bytecode at a time. It exists to keep reference counting and interpreter internals safe without fine-grained locks.

Consequences:

- **I/O-bound threads scale fine.** Blocking calls (`socket.recv`, `time.sleep`, file reads) release the GIL.
- **CPU-bound threads don't.** Two threads counting to 100M take as long as (or longer than) one.
- **C extensions can release the GIL** (NumPy, hashlib, zlib), so they can run in parallel threads.
- **The GIL doesn't make your code thread-safe.** `counter += 1` is several bytecodes; threads can interleave between them.

```python
import threading
import time

def count(n: int = 30_000_000) -> None:
    total = 0
    for _ in range(n):
        total += 1

start = time.perf_counter()
threads = [threading.Thread(target=count) for _ in range(2)]
for t in threads: t.start()
for t in threads: t.join()
print(f"2 threads: {time.perf_counter() - start:.2f}s")   # ≈ same as running count() twice serially
```

**Free-threaded Python:** PEP 703 builds (`python3.13t`, `python3.14t`) remove the GIL. Experimental in 3.13, officially supported (but not the default) since 3.14; 3.15 adds a stable ABI for free-threaded extensions and macOS installers include the build by default. Check with `sys._is_gil_enabled()`. Many C extensions still need to declare support, so I treat it as opt-in for now.

## Threading

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
import Foundation

let group = DispatchGroup()
let queue = DispatchQueue.global()

for name in ["A", "B"] {
    queue.async(group: group) {
        Thread.sleep(forTimeInterval: 0.1)
        print("\(name) done")
    }
}
group.wait()                      // join
```

</td>
<td>

```python
import threading
import time

def worker(name: str) -> None:
    time.sleep(0.1)
    print(f"{name} done")

threads = [threading.Thread(target=worker, args=(n,)) for n in ["A", "B"]]
for t in threads:
    t.start()
for t in threads:
    t.join()                      # ≈ group.wait()
```

</td>
</tr>
</table>

Prefer `ThreadPoolExecutor` (below) over raw threads — it is the closest thing to a dispatch queue with a worker pool.

**Race conditions and locks:**

```python
import threading

counter = 0
lock = threading.Lock()

def increment(times: int) -> None:
    global counter
    for _ in range(times):
        with lock:                # acquire/release automatically
            counter += 1

threads = [threading.Thread(target=increment, args=(100_000,)) for _ in range(8)]
for t in threads: t.start()
for t in threads: t.join()
print(counter)                    # → 800000 (without the lock: often less)
```

Other primitives: `RLock` (re-entrant), `Semaphore`/`BoundedSemaphore` (limit concurrency), `Event` (signal), `Condition`, `Barrier`, and `queue.Queue` (thread-safe producer/consumer — usually the cleanest option).

**Deadlocks** happen when two threads acquire the same locks in opposite order:

```python
import threading

lock_a, lock_b = threading.Lock(), threading.Lock()

def task1():
    with lock_a:
        with lock_b: ...          # waits for B while holding A

def task2():
    with lock_b:
        with lock_a: ...          # waits for A while holding B → deadlock
```

Fixes: always acquire locks in a single global order, hold locks for the shortest possible time, use `lock.acquire(timeout=...)`, or avoid shared state by passing messages through a `Queue`.

**Daemon vs non-daemon threads:**

| | Non-daemon (default) | `daemon=True` |
|---|---|---|
| Program exit | Interpreter waits for the thread to finish | Thread is killed abruptly when the main thread exits |
| Use for | Work that must complete | Heartbeats, monitors, background polling |

⚠️ Daemon threads are terminated without cleanup — no `finally`, no flushed files. Never use them for writes that must complete. A non-daemon thread with `while True:` keeps your program alive forever.

## `concurrent.futures` — the API I actually use

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import time

def fetch(url: str) -> tuple[str, int]:
    time.sleep(0.1)                         # stand-in for a blocking HTTP call
    return url, 200

urls = [f"https://example.com/{i}" for i in range(10)]

with ThreadPoolExecutor(max_workers=5) as pool:
    results = list(pool.map(fetch, urls))   # preserves input order

with ThreadPoolExecutor(max_workers=5) as pool:
    futures = {pool.submit(fetch, u): u for u in urls}
    for future in as_completed(futures):    # yields as each one finishes
        url, status = future.result()       # re-raises the worker's exception here
```

## Multiprocessing

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
// Real parallel CPU work on all cores:
let primeCount = await withTaskGroup(of: Bool.self) { group in
    for n in 1_000_000..<1_000_200 {
        group.addTask { isPrime(n) }      // runs on the thread pool
    }
    return await group.reduce(0) { $0 + ($1 ? 1 : 0) }
}
print(primeCount)
```

</td>
<td>

```python
from concurrent.futures import ProcessPoolExecutor

def is_prime(n: int) -> bool:
    return n > 1 and all(n % d for d in range(2, int(n**0.5) + 1))

if __name__ == "__main__":
    with ProcessPoolExecutor() as pool:       # one process per core
        results = list(pool.map(is_prime, range(1_000_000, 1_000_200)))
    print(sum(results))
```

</td>
</tr>
</table>

Swift gets parallelism from threads. CPU-bound Python gets it from **processes**: each worker is a separate Python with its own memory, so arguments and results are copied between them (pickled).

Each process gets its own interpreter, memory and GIL. Arguments and results are **pickled** across process boundaries, so they must be picklable (top-level functions, not lambdas) and large payloads are expensive to send.

Low-level `multiprocessing` primitives for sharing data:

```python
from multiprocessing import Process, Queue, Value

def produce(q: Queue) -> None:
    q.put("result from child")

def bump(counter) -> None:
    for _ in range(10_000):
        with counter.get_lock():       # Value carries its own lock
            counter.value += 1

if __name__ == "__main__":
    q: Queue = Queue()
    p = Process(target=produce, args=(q,))
    p.start(); print(q.get()); p.join()

    counter = Value("i", 0)            # shared C int
    procs = [Process(target=bump, args=(counter,)) for _ in range(4)]
    for p in procs: p.start()
    for p in procs: p.join()
    print(counter.value)               # → 40000
```

⚠️ Start-method differences: Linux used `fork` by default before 3.14 (now `forkserver`); macOS and Windows use `spawn`, which re-imports your main module in every child. Code outside the `__main__` guard runs once per worker.

## asyncio fundamentals

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
func job(_ name: String, _ seconds: Double) async -> String {
    try? await Task.sleep(for: .seconds(seconds))
    return "\(name) done"
}

@main struct App {
    static func main() async {
        async let a = job("export", 0.3)
        async let b = job("resize", 0.2)
        let results = await [a, b]       // ≈ 0.3s total
        print(results)
    }
}
```

</td>
<td>

```python
import asyncio

async def job(name: str, seconds: float) -> str:
    await asyncio.sleep(seconds)
    return f"{name} done"

async def main() -> None:
    results = await asyncio.gather(     # ≈ async let … await [a, b]
        job("export", 0.3),
        job("resize", 0.2),
    )
    print(results)                      # ≈ 0.3s total

asyncio.run(main())                     # ≈ async main entry point
```

</td>
</tr>
</table>

Calling `job(...)` without `await` doesn't start it — it just creates a coroutine object. `gather` starts them together and waits for all.

- `async def` defines a **coroutine function**. Calling it returns a coroutine object; nothing runs yet.
- `await` suspends the current coroutine until the awaitable completes, letting the **event loop** run other coroutines.
- `asyncio.run(main())` creates the loop, runs `main`, and closes the loop. Call it **once** at the program entry point.
- Everything runs on **one thread**. Concurrency comes from coroutines voluntarily yielding at `await` points.

⚠️ Replacing `await asyncio.sleep(...)` with `time.sleep(...)` makes the example take 0.5s instead of 0.3s — and blocks every other coroutine in the process while it sleeps.

## Tasks and structured concurrency

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
func fetchAll() async throws -> [Int] {
    try await withThrowingTaskGroup(of: Int.self) { group in
        for i in 0..<5 {
            group.addTask { try await fetch(i) }
        }
        var results: [Int] = []
        for try await value in group {     // first error cancels the rest
            results.append(value)
        }
        return results
    }
}

let handle = Task { try await fetchAll() }   // unstructured task
handle.cancel()
```

</td>
<td>

```python
import asyncio

async def fetch(i: int) -> int:
    await asyncio.sleep(0.01 * i)
    return i * 10

async def fetch_all() -> list[int]:
    async with asyncio.TaskGroup() as tg:     # first error cancels the rest
        tasks = [tg.create_task(fetch(i)) for i in range(5)]
    return [t.result() for t in tasks]

async def main() -> None:
    handle = asyncio.create_task(fetch_all())  # unstructured task
    print(await handle)                         # → [0, 10, 20, 30, 40]
    # handle.cancel()

asyncio.run(main())
```

</td>
</tr>
</table>

`TaskGroup` is the direct counterpart of `withThrowingTaskGroup`. One difference: when children fail, Python raises an `ExceptionGroup` containing all the errors (handle it with `except*`).

`asyncio.create_task()` schedules a coroutine to run concurrently and returns a `Task` (a subclass of `Future`). **`asyncio.TaskGroup`** (3.11+) is the structured-concurrency primitive — the equivalent of Swift's `withThrowingTaskGroup`:

```python
import asyncio

async def fetch(i: int) -> int:
    await asyncio.sleep(0.05 * i)
    if i == 3:
        raise ValueError(f"item {i} failed")
    return i * 10

async def main() -> None:
    try:
        async with asyncio.TaskGroup() as tg:          # waits for all children on exit
            tasks = [tg.create_task(fetch(i)) for i in range(5)]
    except* ValueError as eg:                          # first failure cancels the siblings
        print("failed:", [str(e) for e in eg.exceptions])
    else:
        print([t.result() for t in tasks])

asyncio.run(main())
```

| | `gather` | `TaskGroup` |
|---|---|---|
| On first error | Propagates it; **other tasks keep running** | Cancels remaining tasks, raises `ExceptionGroup` |
| Collect errors as values | `return_exceptions=True` | Wrap each coroutine in its own try/except |
| Structured (no orphan tasks) | No | Yes |

⚠️ Keep a reference to fire-and-forget tasks. The loop holds only weak references, so an unreferenced task can be garbage-collected mid-flight:

```python
import asyncio

background: set[asyncio.Task] = set()

def spawn(coro) -> None:
    task = asyncio.create_task(coro)
    background.add(task)
    task.add_done_callback(background.discard)
```

## Timeouts, cancellation, and limiting concurrency

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
func slow() async throws -> String {
    try await Task.sleep(for: .seconds(10))   // throws CancellationError
    return "done"
}

let task = Task { try await slow() }
task.cancel()

do {
    try Task.checkCancellation()
} catch is CancellationError {
    print("cancelled")
}
```

</td>
<td>

```python
import asyncio

async def slow() -> str:
    try:
        await asyncio.sleep(10)
        return "done"
    except asyncio.CancelledError:
        print("cleaning up")
        raise                       # always re-raise

async def main() -> None:
    try:
        async with asyncio.timeout(0.1):
            await slow()
    except TimeoutError:
        print("timed out")

asyncio.run(main())    # → cleaning up / timed out
```

</td>
</tr>
</table>

Both languages cancel cooperatively. Python delivers cancellation as a `CancelledError` raised at the coroutine's current `await`, so `finally` blocks run. Never swallow it.

Limiting how many operations run at once (Swift has no built-in equivalent; you'd cap the number of child tasks you add to a group):

```python
import asyncio

async def main() -> None:
    semaphore = asyncio.Semaphore(5)              # at most 5 in flight
    async def limited(i: int) -> int:
        async with semaphore:
            await asyncio.sleep(0.01)
            return i
    print(sum(await asyncio.gather(*(limited(i) for i in range(50)))))   # → 1225

asyncio.run(main())
```

Cancellation is delivered as `asyncio.CancelledError` at the next `await`. Clean up in `finally` and **re-raise** it; swallowing `CancelledError` breaks timeouts and TaskGroups.

**Swift:** this is the same model as cooperative cancellation via `Task.isCancelled`/`checkCancellation()`, except Python injects the exception for you at the suspension point.

## Mixing blocking code with asyncio

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
// Moving blocking work off the cooperative pool:
let stock = await Task.detached(priority: .utility) {
    blockingLookup("sku-42")
}.value
```

</td>
<td>

```python
import asyncio, time

def blocking_lookup(item: str) -> str:
    time.sleep(0.2)
    return f"{item}: 42"

async def main() -> None:
    stock = await asyncio.to_thread(blocking_lookup, "sku-42")
    print(stock)

asyncio.run(main())
```

</td>
</tr>
</table>

This matters much more in Python than in Swift. One blocking call in a coroutine freezes the **entire** program, not just one thread of a pool.

Never call blocking functions directly in a coroutine. Offload them:

CPU-bound work goes to a process pool instead of a thread:

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor

def cpu_heavy(n: int) -> int:
    return sum(i * i for i in range(n))

async def main() -> None:
    loop = asyncio.get_running_loop()
    with ProcessPoolExecutor() as pool:
        print(await loop.run_in_executor(pool, cpu_heavy, 1_000_000))

if __name__ == "__main__":
    asyncio.run(main())
```

Going the other way — scheduling a coroutine from a regular thread onto a running loop — use `asyncio.run_coroutine_threadsafe(coro, loop)`.

## Futures

A `Future` is a low-level placeholder for a result that will exist later. You rarely create one directly; `Task` is a `Future` that drives a coroutine. `concurrent.futures.Future` (threads/processes) and `asyncio.Future` are different classes — bridge them with `asyncio.wrap_future()`.

## Async iteration and async context managers

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
func ticker(_ n: Int) -> AsyncStream<Int> {
    AsyncStream { continuation in
        Task {
            for i in 0..<n {
                try? await Task.sleep(for: .milliseconds(10))
                continuation.yield(i)
            }
            continuation.finish()
        }
    }
}

for await tick in ticker(3) { print(tick) }
```

</td>
<td>

```python
import asyncio

async def ticker(n: int):                # async generator
    for i in range(n):
        await asyncio.sleep(0.01)
        yield i                          # ≈ continuation.yield

async def main() -> None:
    async for tick in ticker(3):         # ≈ for await
        print(tick)

asyncio.run(main())
```

</td>
</tr>
</table>

An async generator (`async def` + `yield`) replaces `AsyncStream` and its continuation in a few lines.

```python
import asyncio
from collections.abc import AsyncIterator
from contextlib import asynccontextmanager

async def ticker(n: int) -> AsyncIterator[int]:       # async generator
    for i in range(n):
        await asyncio.sleep(0.01)
        yield i

@asynccontextmanager
async def connection(name: str):
    print(f"open {name}")
    try:
        yield name
    finally:
        print(f"close {name}")

async def main() -> None:
    async with connection("db") as conn:
        async for tick in ticker(3):
            print(conn, tick)
        squares = [i * i async for i in ticker(3)]    # async comprehension
        print(squares)

asyncio.run(main())
```

Class form: implement `__aenter__`/`__aexit__` or `__aiter__`/`__anext__`.

## Producer/consumer with `asyncio.Queue`

<table>
<tr><th>Swift (actor)</th><th>Python (lock or queue)</th></tr>
<tr>
<td>

```swift
actor Counter {
    private var value = 0
    func increment() { value += 1 }    // serialised by the actor
    func current() -> Int { value }
}

let counter = Counter()
await withTaskGroup(of: Void.self) { group in
    for _ in 0..<100 { group.addTask { await counter.increment() } }
}
print(await counter.current())         // 100
```

</td>
<td>

```python
import asyncio

class Counter:                            # no actors in Python
    def __init__(self) -> None:
        self._value = 0
        self._lock = asyncio.Lock()       # needed only if an `await`
                                          # sits inside the update
    async def increment(self) -> None:
        async with self._lock:
            current = self._value
            await asyncio.sleep(0)        # a suspension point mid-update
            self._value = current + 1

async def main() -> None:
    c = Counter()
    async with asyncio.TaskGroup() as tg:
        for _ in range(100):
            tg.create_task(c.increment())
    print(c._value)                       # → 100

asyncio.run(main())
```

</td>
</tr>
</table>

Python has no actors. On a single event loop, code between two `await`s can't be interrupted by other coroutines, so most state is safe automatically. Add an `asyncio.Lock` when an update spans an `await`. Passing messages through an `asyncio.Queue` (below) is the other actor-like option.

```python
import asyncio

async def producer(q: asyncio.Queue[int]) -> None:
    for i in range(10):
        await q.put(i)

async def consumer(name: str, q: asyncio.Queue[int], results: list[int]) -> None:
    while True:
        item = await q.get()
        try:
            await asyncio.sleep(0.01)
            results.append(item * 2)
        finally:
            q.task_done()

async def main() -> None:
    q: asyncio.Queue[int] = asyncio.Queue(maxsize=5)     # bounded → backpressure
    results: list[int] = []
    workers = [asyncio.create_task(consumer(f"w{i}", q, results)) for i in range(3)]
    await producer(q)
    await q.join()                                        # wait until every item is processed
    for w in workers:
        w.cancel()
    await asyncio.gather(*workers, return_exceptions=True)
    print(sorted(results))

asyncio.run(main())
```

## Async HTTP

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
import Foundation

func statusCodes(_ urls: [URL]) async throws -> [Int] {
    try await withThrowingTaskGroup(of: Int.self) { group in
        for url in urls {
            group.addTask {
                let (_, resp) = try await URLSession.shared.data(from: url)
                return (resp as! HTTPURLResponse).statusCode
            }
        }
        return try await group.reduce(into: []) { $0.append($1) }
    }
}
```

</td>
<td>

```python
import asyncio
import httpx

async def status_codes(urls: list[str]) -> list[int]:
    async with httpx.AsyncClient(timeout=10) as client:   # ≈ URLSession
        responses = await asyncio.gather(*(client.get(u) for u in urls))
        return [r.status_code for r in responses]

# asyncio.run(status_codes(["https://example.com"] * 3))
```

</td>
</tr>
</table>

`aiohttp` is the other mainstream option (`async with aiohttp.ClientSession() as session: async with session.get(url) as resp: ...`). Either way: create the session once and share it.

## Swift Concurrency ↔ asyncio

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

**Python has no actors.** Within a single event loop, code between two `await`s is atomic with respect to other coroutines — so many races vanish — but any `await` in the middle of a read-modify-write is a suspension point where another coroutine can interleave. Use `asyncio.Lock` there. And `asyncio.Lock` is **not** thread-safe; it only coordinates coroutines on one loop.

---

[← 09. File Handling and Data Serialization](09-files-and-serialization.md) · [Contents](../README.md) · [11. Testing and Code Quality →](11-testing-and-quality.md)
