# 11. Testing and Code Quality

[← 10. Concurrency and Asynchronous Programming](10-concurrency.md) · [Contents](../README.md) · [12. Python Standard Library →](12-standard-library.md)

> **In Swift terms:** pytest plays the role of Swift Testing (`@Test`, `#expect`): plain functions, plain `assert`, built-in parameterised tests. `unittest` is the XCTest-style framework. Ruff is SwiftLint + swift-format; Pyright/mypy are the type checker you lose by leaving the Swift compiler.

## pytest

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
import Testing
@testable import Pricing

@Test func appliesDiscount() {
    #expect(applyDiscount(200, 10) == 180)
}

@Test(arguments: [(100.0, 0.0, 100.0), (100, 50, 50)])
func discountCases(total: Double, percent: Double, expected: Double) {
    #expect(applyDiscount(total, percent) == expected)
}

@Test func rejectsInvalidPercent() {
    #expect(throws: PricingError.self) {
        try validatedDiscount(100, 120)
    }
}
```

</td>
<td>

```python
# tests/test_pricing.py
import pytest
# from pricing import apply_discount

def apply_discount(total: float, percent: float) -> float:   # stand-in
    if not 0 <= percent <= 100:
        raise ValueError("percent must be within 0..100")
    return total * (1 - percent / 100)

def test_applies_discount() -> None:
    assert apply_discount(200, 10) == 180

@pytest.mark.parametrize(("total", "percent", "expected"),
                         [(100, 0, 100), (100, 50, 50)])
def test_discount_cases(total, percent, expected) -> None:
    assert apply_discount(total, percent) == expected

def test_rejects_invalid_percent() -> None:
    with pytest.raises(ValueError):
        apply_discount(100, 120)
```

</td>
</tr>
</table>

Test discovery is by name: files `test_*.py`, functions `test_*`. Run with `pytest`.

Two assertions you'll use constantly:

```python
import pytest

def test_float_result() -> None:
    assert 0.1 + 0.2 == pytest.approx(0.3)          # like #expect(abs(a - b) < ε)

def test_error_message() -> None:
    with pytest.raises(ValueError, match="invalid literal"):   # regex on the message
        int("x")
```

```bash
pytest -q                     # run everything
pytest tests/test_pricing.py::test_apply_discount -x   # one test, stop at first failure
pytest -k "discount and not invalid" -vv
pytest --lf                   # rerun last failures
pytest --cov=src              # coverage (pytest-cov)
```

## Fixtures

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
import Testing

@Suite struct RepositoryTests {
    let repo: MemoryRepository

    init() {                     // fresh setup for every test
        repo = MemoryRepository()
        repo.save("a", ["name": "Ada"])
    }

    @Test func findsUser() {
        #expect(repo.get("a") != nil)
    }
}
```

</td>
<td>

```python
import pytest

@pytest.fixture
def repo() -> dict[str, dict]:      # fresh setup for every test
    return {"a": {"name": "Ada"}}

def test_finds_user(repo: dict[str, dict]) -> None:   # injected by NAME
    assert "a" in repo
```

</td>
</tr>
</table>

A fixture is injected into any test that has a parameter with the fixture's name. `yield` inside a fixture separates setup from teardown.

Fixtures are dependency injection for tests. Put shared ones in `conftest.py`.

```python
import pytest
from pathlib import Path

@pytest.fixture
def sample_file(tmp_path: Path) -> Path:          # tmp_path: built-in per-test temp dir
    path = tmp_path / "data.csv"
    path.write_text("name\nAda\n", encoding="utf-8")
    return path

@pytest.fixture(scope="session")
def db():
    conn = {"connected": True}                    # expensive setup once per test run
    yield conn                                    # code after yield = teardown
    conn["connected"] = False

def test_reads_file(sample_file: Path) -> None:
    assert sample_file.read_text().startswith("name")
```

Useful built-ins: `tmp_path`, `monkeypatch` (patch env vars/attributes), `capsys` (captured stdout), `caplog` (captured logs).

```python
def test_reads_env(monkeypatch: pytest.MonkeyPatch) -> None:
    monkeypatch.setenv("API_URL", "http://test")
    import os
    assert os.environ["API_URL"] == "http://test"
```

## Mocking

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
import Testing

protocol UserClient {
    func fetchName(id: Int) async throws -> String
}

struct FakeClient: UserClient {
    func fetchName(id: Int) async throws -> String { "Ada" }
}

@Test func greets() async throws {
    let service = Greeter(client: FakeClient())
    #expect(try await service.greet(id: 1) == "Hi Ada")
}
```

</td>
<td>

```python
from unittest.mock import Mock

def greet(client, user_id: int) -> str:
    return f"Hi {client.fetch_name(user_id)}"

def test_greets() -> None:
    client = Mock()                         # accepts ANY method call
    client.fetch_name.return_value = "Ada"
    assert greet(client, 1) == "Hi Ada"
    client.fetch_name.assert_called_once_with(1)
```

</td>
</tr>
</table>

Protocol-based fakes work in Python too, and I still prefer them. `Mock` objects are the dynamic shortcut: they accept any attribute or call and record what happened.

When you can't inject a dependency, `patch` swaps a name for the duration of a test. Patch it where it is **looked up**, not where it is defined:

```python
from unittest.mock import patch

# with patch("my_service.api.httpx.get") as fake_get:
#     fake_get.return_value.json.return_value = {"name": "Ada"}
#     ...
```

`AsyncMock` for coroutines; `pytest-asyncio` or `anyio` to run `async def` tests; `respx` (httpx) or `responses` (requests) to mock HTTP at the transport layer.

## unittest

The standard library framework (xUnit-style, like XCTest). pytest runs unittest tests unchanged, so the main reason to know it is existing codebases.

```python
import unittest

class TestPricing(unittest.TestCase):
    def setUp(self) -> None:
        self.total = 100
    def test_half(self) -> None:
        self.assertEqual(self.total * 0.5, 50)
    def test_raises(self) -> None:
        with self.assertRaises(ZeroDivisionError):
            1 / 0

if __name__ == "__main__":
    unittest.main()
```

**Swift:** pytest feels closer to Swift Testing (`@Test`, `#expect`, parameterised `arguments:`) than to XCTest.

## Formatting, linting, static analysis

| Tool | Role |
|---|---|
| **Ruff** | Linter *and* formatter (replaces Flake8, isort, pyupgrade, Black for most teams). Very fast |
| Black | Formatter (Ruff's formatter is compatible) |
| **Pyright** / **mypy** | Type checking |
| Bandit | Security-focused lint |
| pre-commit | Runs the above on every commit |

```bash
ruff check . --fix
ruff format .
pyright
```

```toml
[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "SIM", "RUF", "ASYNC", "S"]
```

`B` (bugbear) catches mutable defaults and loop-variable closures; `ASYNC` catches blocking calls inside coroutines; `S` flags `eval`, `shell=True`, hard-coded passwords.

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.6.9
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
```

---

[← 10. Concurrency and Asynchronous Programming](10-concurrency.md) · [Contents](../README.md) · [12. Python Standard Library →](12-standard-library.md)
