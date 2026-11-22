# 07. Modules, Packages and Dependency Management

[← 06. Error Handling and Debugging](06-errors-and-debugging.md) · [Contents](../README.md) · [08. Functional Programming and Pythonic Patterns →](08-functional-patterns.md)

> **In Swift terms:** a `.py` file is a module and a folder of them is a package — there are no targets or build steps. A virtual environment is a private copy of Python with its own installed packages, one per project. `pyproject.toml` is your `Package.swift`.

## Import system

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
import Foundation             // whole module, names unqualified
// Names from an imported module are global in Swift:
let url = URL(string: "https://x.dev")
```

</td>
<td>

```python
import json                          # module object; use json.loads
from pathlib import Path             # bring one name in
from collections import defaultdict as dd

data = json.loads("{}")              # names stay qualified by default
p = Path("x")
```

</td>
</tr>
</table>

Swift imports dump every public name into your file. Python imports give you a **module object**; you either write `module.name` or import names explicitly. Each file has its own namespace.

A **module** is a `.py` file. A **package** is a directory of modules (with `__init__.py` for a regular package). The first import executes the module top to bottom and caches it in `sys.modules`; later imports reuse it.

```python
import json                              # module object
from pathlib import Path                 # a name from a module
from collections import defaultdict as dd
from .utils import slugify               # relative import (inside a package only)
from ..core.models import Order
```

Imports are resolved via `sys.path`: the script's directory (or CWD for `-m`), `PYTHONPATH`, then site-packages of the active environment.

⚠️ Don't use `from module import *` in application code; it hides where names come from.

## `__init__.py`

- Marks a directory as a regular package and runs on first import of the package.
- Use it to define the public API (`from .client import Client`, `__all__ = [...]`) — keep it light; heavy work here slows every import.
- Without it you get a *namespace package*, which works but is meant for packages split across multiple directories.

## `if __name__ == "__main__":`

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
// main.swift (or @main) is the entry point.
// Other files never execute top-level code.
@main
struct App {
    static func main() { print("run") }
}
```

</td>
<td>

```python
def main() -> int:
    print("run")
    return 0

if __name__ == "__main__":     # true only when run directly,
    raise SystemExit(main())   # false when imported
```

</td>
</tr>
</table>

In Python, importing a file **executes** it from top to bottom. The `__main__` check keeps the "run the program" part from running when another file imports this one.

`__name__` is `"__main__"` when a file is executed directly and the module's dotted name when imported. Guard script entry points so importing the module (from tests, or by `multiprocessing` on macOS/Windows, which re-imports the main module in child processes) doesn't run them:

## Project layout

```text
my-service/
├── pyproject.toml
├── README.md
├── src/
│   └── my_service/
│       ├── __init__.py
│       ├── __main__.py          # enables `python -m my_service`
│       ├── config.py
│       ├── api/
│       ├── domain/
│       └── adapters/
├── tests/
│   ├── conftest.py
│   └── test_domain.py
└── .env.example
```

The `src/` layout prevents tests from accidentally importing the uninstalled working copy. Run code as modules (`python -m my_service`) so absolute imports resolve the same way everywhere.

## Virtual environments

Every project gets its own isolated interpreter + site-packages. Never `pip install` into the system Python.

```bash
python3 -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
python -m pip install requests     # `python -m pip` guarantees the venv's pip
deactivate
```

Add `.venv/` to `.gitignore`. Point your editor at `.venv/bin/python` so autocompletion and type checking use the right packages.

## Tooling options

| Tool | What it is | When I use it |
|---|---|---|
| `venv` + `pip` | Standard library + default installer | Minimal setups, Docker images |
| **uv** | Fast Rust-based replacement for pip, venv, pip-tools, pipx and pyenv; lockfiles | My default for new projects |
| Poetry / PDM / Hatch | Project managers with lockfiles and build backends | Existing projects that use them |
| Conda / Mamba | Environments that include non-Python binaries (CUDA, MKL, GDAL) | Data science / GPU stacks with native deps |
| pipx / `uv tool` | Install CLI tools globally in isolated envs | `ruff`, `pre-commit`, `httpie` |

```bash
uv init my-service && cd my-service
uv add httpx "pydantic>=2.7"        # updates pyproject.toml and uv.lock
uv add --dev pytest ruff pyright
uv run pytest                       # runs inside the project env, auto-syncs
uv python install 3.13              # manage interpreter versions
```

## `pyproject.toml`

<table>
<tr><th>Swift</th><th>Python</th></tr>
<tr>
<td>

```swift
// Package.swift
// swift-tools-version: 6.0
import PackageDescription

let package = Package(
    name: "MyService",
    platforms: [.macOS(.v14)],
    dependencies: [
        .package(url: "https://github.com/apple/swift-log",
                 from: "1.5.0"),
    ],
    targets: [
        .executableTarget(name: "MyService",
            dependencies: [.product(name: "Logging",
                                    package: "swift-log")]),
    ]
)
```

</td>
<td>

```python
# pyproject.toml (TOML, shown as comments here)
# [project]
# name = "my-service"
# version = "0.1.0"
# requires-python = ">=3.12"        ≈ platforms / tools version
# dependencies = [
#     "httpx>=0.27",                 ≈ .package(from:)
# ]
#
# [project.scripts]
# my-service = "my_service.__main__:main"   ≈ executableTarget
```

</td>
</tr>
</table>

| SPM | Python |
|---|---|
| `swift package resolve` | `uv lock` / `uv sync` |
| `swift build` | nothing to build; `uv sync` installs deps |
| `swift run` | `uv run python -m my_service` |
| `swift test` | `uv run pytest` |
| `Package.resolved` | `uv.lock` |
| `.build/` | `.venv/` |

The single configuration file for metadata, dependencies, and tools (PEP 518/621).

```toml
[project]
name = "my-service"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "httpx>=0.27",
    "pydantic>=2.7,<3",
]

[project.optional-dependencies]
data = ["polars>=1.0", "pyarrow>=16"]

[project.scripts]
my-service = "my_service.__main__:main"   # installs a `my-service` CLI command

[dependency-groups]                       # PEP 735 dev dependencies (uv, pip 25.1+)
dev = ["pytest>=8", "ruff>=0.6", "pyright>=1.1.380"]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.ruff]
line-length = 100
target-version = "py312"

[tool.pytest.ini_options]
testpaths = ["tests"]
```

## `requirements.txt`

Still common for deployments and quick scripts:

```bash
pip install -r requirements.txt
pip freeze > requirements.txt      # pins EVERYTHING installed, including transitive deps
```

`pip freeze` output is a lockfile, not a dependency declaration. Declare direct dependencies with ranges in `pyproject.toml`; generate exact pins with `uv lock` / `uv pip compile` / `pip-tools`.

**Swift:** `pyproject.toml` ≈ `Package.swift`; a lockfile (`uv.lock`, `poetry.lock`) ≈ `Package.resolved`; a venv has no SPM equivalent because SPM resolves per project automatically.

## Lazy imports (3.15+)

Python 3.15 adds explicit lazy imports (PEP 810), deferring module loading until first use — useful for CLI start-up time. On older versions, import heavy modules inside the function that needs them.

---

[← 06. Error Handling and Debugging](06-errors-and-debugging.md) · [Contents](../README.md) · [08. Functional Programming and Pythonic Patterns →](08-functional-patterns.md)
