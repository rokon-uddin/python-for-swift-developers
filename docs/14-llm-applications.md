# 14. Python for LLM and AI Application Development

[← 13. Python for Data Engineering and AI](13-data-engineering.md) · [Contents](../README.md) · [15. Common Pitfalls and Best Practices →](15-pitfalls.md)

> **In Swift terms:** an LLM API is just a JSON-over-HTTPS service. Everything here maps to patterns you already use in an iOS networking layer: a configured client, `Codable` request/response models (Pydantic here), a protocol in front of the vendor SDK, retries, streaming, and a server (FastAPI instead of Vapor).

This section is about the *programming* around model APIs: clients, configuration, structured output, retries, streaming, concurrency and service boundaries.

## Configuration and secrets

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
import Foundation

struct Settings {
    let apiKey: String
    let model: String

    static func load() throws -> Settings {
        let env = ProcessInfo.processInfo.environment
        guard let key = env["OPENAI_API_KEY"] else {
            throw ConfigError.missing("OPENAI_API_KEY")
        }
        return Settings(apiKey: key,
                        model: env["MODEL_NAME"] ?? "gpt-4.1-mini")
    }
}
```

</td>
<td>

```python
import os
from dataclasses import dataclass

@dataclass(frozen=True)
class Settings:
    api_key: str
    model: str

    @classmethod
    def load(cls) -> "Settings":
        try:
            key = os.environ["OPENAI_API_KEY"]
        except KeyError:
            raise RuntimeError("missing OPENAI_API_KEY") from None
        return cls(api_key=key, model=os.getenv("MODEL_NAME", "gpt-4.1-mini"))
```

</td>
</tr>
</table>

The hand-written version on the right is fine for small tools. The `pydantic-settings` version below does the same thing declaratively, adds type conversion and `.env` support, and hides the key in logs.

```python
from functools import lru_cache
from pydantic import SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_file_encoding="utf-8", extra="ignore")

    openai_api_key: SecretStr                 # read from OPENAI_API_KEY; never printed in repr/logs
    model_name: str = "gpt-4.1-mini"
    request_timeout: float = 30.0
    max_concurrency: int = 8
    redis_url: str = "redis://localhost:6379/0"

@lru_cache
def get_settings() -> Settings:               # one instance, created lazily, easy to override in tests
    return Settings()

# key = get_settings().openai_api_key.get_secret_value()
```

`python-dotenv`'s `load_dotenv()` is the lightweight alternative: call it once at startup, then read `os.environ`. SDK clients such as `OpenAI()` pick up `OPENAI_API_KEY` from the environment automatically — don't pass keys as literals.

## API clients

```python
from openai import OpenAI, AsyncOpenAI

client = OpenAI(timeout=30, max_retries=2)    # create once, reuse (connection pool)

resp = client.chat.completions.create(
    model="gpt-4.1-mini",
    messages=[
        {"role": "system", "content": "You answer only maths questions."},
        {"role": "user", "content": "Expand (a + b)^2"},
    ],
)
print(resp.choices[0].message.content)
```

Many providers (Gemini, local Ollama, vLLM, OpenRouter) expose an **OpenAI-compatible** endpoint, so the same client works by changing `base_url` and the key:

```python
import os
from openai import OpenAI

gemini = OpenAI(
    api_key=os.environ["GEMINI_API_KEY"],
    base_url="https://generativelanguage.googleapis.com/v1beta/openai/",
)
local = OpenAI(api_key="ollama", base_url="http://localhost:11434/v1")
```

Messages are plain lists of dicts — the conversation history *is* your state. For multimodal input, `content` becomes a list of parts (`{"type": "text", ...}`, `{"type": "image_url", "image_url": {"url": ...}}`).

## Structured outputs with Pydantic

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
import Foundation

struct AgentStep: Codable {
    enum Kind: String, Codable { case plan = "PLAN", tool = "TOOL", output = "OUTPUT" }
    let step: Kind
    let content: String?
    let tool: String?
}

let step = try JSONDecoder().decode(AgentStep.self, from: Data(modelReply.utf8))
```

</td>
<td>

```python
from typing import Literal
from pydantic import BaseModel

class Step(BaseModel):
    step: Literal["PLAN", "TOOL", "OUTPUT"]   # ≈ String-backed enum
    content: str | None = None
    tool: str | None = None

model_reply = '{"step": "TOOL", "tool": "get_weather"}'
step = Step.model_validate_json(model_reply)
print(step.step, step.tool)                    # → TOOL get_weather
```

</td>
</tr>
</table>

Same idea as decoding a server response into a `Codable` struct, except you can also hand the model class to the SDK so the model is constrained to produce that shape.

Asking a model for "JSON" and calling `json.loads` works until it doesn't. Define the schema as a Pydantic model and let the SDK enforce and parse it:

```python
from typing import Literal
from pydantic import BaseModel, Field

class AgentStep(BaseModel):
    step: Literal["PLAN", "TOOL", "OUTPUT"]
    content: str | None = Field(None, description="Reasoning or final answer")
    tool: str | None = Field(None, description="Tool name when step == TOOL")
    tool_input: str | None = None

# completion = client.chat.completions.parse(            # older SDKs: client.beta.chat.completions.parse
#     model="gpt-4.1", messages=history, response_format=AgentStep,
# )
# step = completion.choices[0].message.parsed            # an AgentStep instance (or None on refusal)
```

When a provider lacks native structured output, validate yourself and retry on failure:

```python
from pydantic import ValidationError

def parse_step(raw: str) -> AgentStep | None:
    try:
        return AgentStep.model_validate_json(raw)
    except ValidationError:
        return None          # caller re-prompts with the validation error appended
```

## A minimal tool-calling loop

The core of most "agents" is a loop: ask the model, if it requests a tool then run it and feed the result back, stop when it produces a final answer. The engineering concerns are all Python ones — dispatch tables, validation, bounded iterations, and safety.

```python
from collections.abc import Callable

def get_weather(city: str) -> str:
    return f"{city}: 31°C, humid"            # real version: httpx call with timeout

TOOLS: dict[str, Callable[[str], str]] = {"get_weather": get_weather}

def run_agent(ask_model: Callable[[list[dict]], AgentStep], question: str, max_steps: int = 8) -> str:
    history: list[dict] = [{"role": "user", "content": question}]
    for _ in range(max_steps):                               # never loop unbounded
        step = ask_model(history)
        history.append({"role": "assistant", "content": step.model_dump_json()})
        match step.step:
            case "OUTPUT":
                return step.content or ""
            case "TOOL":
                fn = TOOLS.get(step.tool or "")
                result = fn(step.tool_input or "") if fn else f"unknown tool {step.tool!r}"
                history.append({"role": "user", "content": f"OBSERVATION: {result}"})
            case "PLAN":
                continue
    raise RuntimeError("agent did not finish within max_steps")

# Scripted fake model — the same shape as a real one, which is what makes this testable:
script = iter([
    AgentStep(step="PLAN", content="need weather"),
    AgentStep(step="TOOL", tool="get_weather", tool_input="Dhaka"),
    AgentStep(step="OUTPUT", content="It's 31°C in Dhaka."),
])
print(run_agent(lambda history: next(script), "Weather in Dhaka?"))
```

⚠️ Never give a model a tool that runs arbitrary shell commands (`os.system(cmd)`) on a machine you care about. Allow-list commands, run them with `subprocess.run([...], shell=False, timeout=...)`, sandbox them in a container, and require confirmation for anything destructive. Model output is untrusted input.

Native tool calling (`tools=[...]` with JSON-schema function definitions, `tool_calls` in the response) is more robust than a prompt-defined protocol; frameworks like LangGraph or the providers' agent SDKs wrap the same loop with persistence and streaming.

## Async requests and concurrency limits

```python
import asyncio
from openai import AsyncOpenAI

async def summarise_all(texts: list[str], limit: int = 8) -> list[str]:
    client = AsyncOpenAI()
    sem = asyncio.Semaphore(limit)                 # respect provider rate limits

    async def one(text: str) -> str:
        async with sem:
            r = await client.chat.completions.create(
                model="gpt-4.1-mini",
                messages=[{"role": "user", "content": f"Summarise in one line:\n{text}"}],
            )
            return r.choices[0].message.content or ""

    async with asyncio.TaskGroup() as tg:
        tasks = [tg.create_task(one(t)) for t in texts]
    return [t.result() for t in tasks]
```

## Retries with exponential backoff

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
import Foundation

func withRetries<T>(attempts: Int = 5,
                    _ op: () async throws -> T) async throws -> T {
    for attempt in 1... {
        do {
            return try await op()
        } catch where attempt < attempts {
            let delay = min(20, 0.5 * pow(2, Double(attempt - 1)))
            try await Task.sleep(for: .seconds(delay * .random(in: 0.5...1)))
        }
    }
    fatalError("unreachable")
}
```

</td>
<td>

```python
import random, time

def with_retries(fn, attempts: int = 5):
    for attempt in range(1, attempts + 1):
        try:
            return fn()
        except ConnectionError:
            if attempt == attempts:
                raise
            delay = min(20, 0.5 * 2 ** (attempt - 1))
            time.sleep(delay * random.uniform(0.5, 1))

print(with_retries(lambda: "ok"))    # → ok
```

</td>
</tr>
</table>

In practice I use `tenacity`:

```python
from tenacity import retry, stop_after_attempt, wait_random_exponential, retry_if_exception_type
import httpx

@retry(
    retry=retry_if_exception_type((httpx.TimeoutException, httpx.HTTPStatusError)),
    wait=wait_random_exponential(multiplier=0.5, max=20),
    stop=stop_after_attempt(5),
    reraise=True,
)
def call_api(client: httpx.Client, url: str) -> dict:
    r = client.get(url)
    r.raise_for_status()
    return r.json()
```

Retry only idempotent operations and transient failures (timeouts, 429, 5xx). Don't retry 400/401/422 — they won't fix themselves. Honour `Retry-After` headers when present.

## Streaming responses

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
import Foundation

let (bytes, _) = try await URLSession.shared.bytes(for: request)
for try await line in bytes.lines where line.hasPrefix("data: ") {
    print(line.dropFirst(6), terminator: "")
}
```

</td>
<td>

```python
import httpx

async def stream_lines(url: str) -> None:
    async with httpx.AsyncClient(timeout=None) as client:
        async with client.stream("GET", url) as response:
            async for line in response.aiter_lines():
                if line.startswith("data: "):
                    print(line[6:], end="", flush=True)
```

</td>
</tr>
</table>

```python
# Sync
# with client.chat.completions.create(model=..., messages=..., stream=True) as stream:
#     for chunk in stream:
#         delta = chunk.choices[0].delta.content if chunk.choices else None
#         if delta:
#             print(delta, end="", flush=True)

# Async → re-emit as Server-Sent Events from FastAPI
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()

async def token_stream(prompt: str):
    for word in f"echo: {prompt}".split():       # stand-in for `async for chunk in await client...`
        yield f"data: {word}\n\n"

@app.get("/stream")
async def stream(prompt: str) -> StreamingResponse:
    return StreamingResponse(token_stream(prompt), media_type="text/event-stream")
```

## Serving: FastAPI

<table>
<tr><th>Swift (Vapor)</th><th>Python (FastAPI)</th></tr>
<tr>
<td>

```swift
import Vapor

struct ChatRequest: Content { let message: String }
struct ChatResponse: Content { let reply: String }

func routes(_ app: Application) throws {
    app.post("chat") { req async throws -> ChatResponse in
        let body = try req.content.decode(ChatRequest.self)
        guard !body.message.isEmpty else {
            throw Abort(.unprocessableEntity)
        }
        return ChatResponse(reply: "echo: \(body.message)")
    }
}
```

</td>
<td>

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()

class ChatRequest(BaseModel):
    message: str

class ChatResponse(BaseModel):
    reply: str

@app.post("/chat")
async def chat(req: ChatRequest) -> ChatResponse:   # body decoded & validated
    if not req.message:
        raise HTTPException(status_code=422, detail="empty message")
    return ChatResponse(reply=f"echo: {req.message}")
```

</td>
</tr>
</table>

FastAPI reads the type hints: a Pydantic parameter means "decode the JSON body into this", the return type defines the response schema, and interactive API docs are generated at `/docs`.

FastAPI's `Depends` is its dependency-injection hook — the place to hand a handler its `ChatModel`, and to swap in a fake during tests:

```python
from fastapi import Depends, FastAPI

app = FastAPI()

def get_llm():
    return lambda text: f"echo: {text}"          # production: return OpenAIChat(...)

@app.post("/echo")
async def echo(message: str, llm=Depends(get_llm)) -> dict[str, str]:
    return {"reply": llm(message)}

# In tests: app.dependency_overrides[get_llm] = lambda: FakeChat("ok")
# Run:      uvicorn app:app --reload   (or `fastapi dev app.py`)
```

Rules for async endpoints: `async def` handlers must only `await` non-blocking calls. If a dependency is synchronous (a sync SDK, `requests`, a sync DB driver), declare the handler with plain `def` — FastAPI runs it in a thread pool — or wrap the call in `asyncio.to_thread`.

## Offloading slow work to a queue

LLM calls and document indexing take seconds to minutes; HTTP requests shouldn't wait. Pattern: enqueue → return a job ID → client polls (or receives a webhook/SSE).

```python
# client.py
# from redis import Redis
# from rq import Queue
# queue = Queue(connection=Redis.from_url("redis://localhost:6379/0"))

# server.py
# @app.post("/jobs")
# def submit(query: str):
#     job = queue.enqueue("worker.process_query", query, job_timeout=300, result_ttl=3600)
#     return {"job_id": job.id, "status": job.get_status()}
#
# @app.get("/jobs/{job_id}")
# def status(job_id: str):
#     job = queue.fetch_job(job_id)
#     if job is None:
#         raise HTTPException(404, "unknown job")
#     return {"status": job.get_status(), "result": job.return_value()}   # None until finished

# run workers (scale horizontally): `rq worker` — on macOS: `rq worker --worker-class rq.SimpleWorker` if fork causes issues
```

Job functions must be importable by the worker (module path, not a lambda) and their arguments picklable. Valkey is a drop-in Redis replacement. For heavier needs (retries, scheduling, chaining), Celery, Dramatiq or Arq (asyncio-native) are the usual choices.

## Reusable service abstractions

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
protocol ChatModel {
    func complete(_ messages: [Message]) async throws -> String
}

struct FakeChat: ChatModel {
    var reply = "ok"
    func complete(_ messages: [Message]) async throws -> String { reply }
}

func answer(_ q: String, using llm: some ChatModel) async throws -> String {
    try await llm.complete([Message(role: "user", content: q)])
}
```

</td>
<td>

```python
from typing import Protocol
from dataclasses import dataclass

class ChatModel(Protocol):
    def complete(self, messages: list[dict[str, str]]) -> str: ...

@dataclass
class FakeChat:                       # conforms by shape
    reply: str = "ok"
    def complete(self, messages: list[dict[str, str]]) -> str:
        return self.reply

def answer(q: str, llm: ChatModel) -> str:
    return llm.complete([{"role": "user", "content": q}])

print(answer("hi", FakeChat("42")))   # → 42
```

</td>
</tr>
</table>

Exactly the protocol-oriented dependency injection you'd use in an iOS app.

Hide vendor SDKs behind a `Protocol` so the rest of the code — and the tests — never import them:

The production side: an implementation backed by the vendor SDK, plus a second protocol for embeddings. Nothing else in the codebase imports `openai`.

```python
from dataclasses import dataclass
from typing import Protocol
from openai import OpenAI

class EmbeddingModel(Protocol):
    def embed(self, texts: list[str]) -> list[list[float]]: ...

@dataclass
class OpenAIChat:                     # conforms to ChatModel by shape
    client: OpenAI
    model: str = "gpt-4.1-mini"

    def complete(self, messages: list[dict[str, str]]) -> str:
        r = self.client.chat.completions.create(model=self.model, messages=messages)
        return r.choices[0].message.content or ""
```

## Retrieval-augmented generation — the Python shape

Indexing and retrieval are two separate programs:

```python
def chunk_text(text: str, size: int = 1000, overlap: int = 200) -> list[str]:
    """Fixed-size character chunks with overlap so sentences at boundaries aren't lost."""
    if overlap >= size:
        raise ValueError("overlap must be smaller than size")
    step = size - overlap
    return [text[i : i + size] for i in range(0, max(len(text) - overlap, 1), step)]

print([len(c) for c in chunk_text("x" * 2500)])   # → [1000, 1000, 900]
```

- **Indexing:** load documents (e.g. `PyPDFLoader`) → split (`RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)`) → embed → upsert into a vector store (Qdrant, pgvector, Chroma) with metadata like `source` and `page`.
- **Retrieval:** embed the query → similarity search (top-k) → format chunks with their metadata into the prompt → generate.
- Keep the embedding model identical between indexing and querying, store enough metadata to cite sources, and batch embedding calls.

## Stateful graphs and persistence

Agent frameworks such as LangGraph model a workflow as typed state flowing through nodes:

```python
from typing import Annotated, TypedDict
import operator

class State(TypedDict):
    messages: Annotated[list[str], operator.add]   # "reducer": node outputs are appended, not replaced
    route: str | None

# graph = StateGraph(State)
# graph.add_node("chatbot", chatbot); graph.add_node("review", review)
# graph.add_edge(START, "chatbot")
# graph.add_conditional_edges("chatbot", pick_next)    # pick_next(state) -> "review" | END
# app = graph.compile(checkpointer=MongoDBSaver(...))  # persists state per thread_id
# app.invoke({"messages": ["hi"]}, config={"configurable": {"thread_id": "user-123"}})
```

The Python takeaways: state is a `TypedDict`, `Annotated` carries the merge function, nodes are plain functions returning **partial** state updates (don't mutate the input), routing functions return a `Literal` of node names, and a checkpointer keyed by `thread_id` turns an in-memory run into resumable conversation memory.

## Local models

- **Ollama** serves local models over HTTP (`ollama` Python client, or the OpenAI-compatible `/v1` endpoint). Run it in Docker with a mounted volume for model weights.
- **Hugging Face `transformers`**: `pipeline("text-generation", model=...)` for quick experiments; gated models need `hf auth login` (formerly `huggingface-cli login`) and an access token in `HF_TOKEN`.
- **Tokenisation:** `tiktoken` (`tiktoken.encoding_for_model("gpt-4o").encode(text)`) to count tokens before sending — essential for context-window budgeting and cost estimates.

## Checklist for LLM-backed services

- One shared client per process; explicit timeouts; bounded retries with jitter.
- Semaphore-limited concurrency matching provider rate limits.
- Pydantic models for every request, response and model output.
- Secrets from the environment / secret manager; `SecretStr`; nothing in source control.
- Streaming for interactive UIs; queues for anything slow.
- Bounded agent loops; allow-listed tools; treat model output as untrusted.
- Log prompts/responses with IDs (and redact PII) so failures are reproducible.

---

[← 13. Python for Data Engineering and AI](13-data-engineering.md) · [Contents](../README.md) · [15. Common Pitfalls and Best Practices →](15-pitfalls.md)
