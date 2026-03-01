---
name: python
description: 'Deep knowledge for writing, reviewing, and debugging Python code (3.9+). Covers PEP 8 style, type hints, error handling, virtual environments, testing, packaging, and common idioms. Load when the user is working with .py files, Python scripts, modules, packages, or Python-based automation.'
---

# Python Scripting & Development Skill

## Scope

This skill applies to **Python 3.9+**. When a project requires support for an older version, note incompatibilities explicitly. The skill covers scripting, CLI tools, automation, and general application code — not framework-specific domains (Django, FastAPI, etc.) unless explicitly combined with a framework skill.

---

## Script Header Requirements

For standalone scripts:

```python
#!/usr/bin/env python3
"""
Short one-line description of what the script does.

Usage:
    python script.py [OPTIONS] <input>
"""
from __future__ import annotations
```

- `#!/usr/bin/env python3` — portable shebang.
- `from __future__ import annotations` — defers annotation evaluation (enables forward references; standard in 3.10+ but safe to add for 3.9).
- Docstring at module level, following PEP 257.

---

## Style Guide (PEP 8 + PEP 257)

- 4-space indentation. Never tabs.
- Max line length: **88 characters** (Black default) — stricter than PEP 8's 79.
- Two blank lines between top-level definitions; one blank line between methods.
- `snake_case` for variables, functions, modules. `PascalCase` for classes. `UPPER_SNAKE_CASE` for module-level constants.
- Docstrings: triple double-quotes `"""`. Google or NumPy style for multi-section docs.
- Avoid `from module import *` — always name imports explicitly.
- Group imports in this order (separated by blank lines):
  1. Standard library
  2. Third-party
  3. Local / project imports

---

## Type Hints (PEP 484 / 526 / 604)

Always add type hints to function signatures and class attributes.

```python
from __future__ import annotations
from collections.abc import Sequence
from pathlib import Path

def process_files(
    paths: Sequence[Path],
    *,
    verbose: bool = False,
    max_workers: int = 4,
) -> dict[str, int]:
    ...
```

- Use `X | None` (3.10+ syntax, safe with `from __future__ import annotations` on 3.9).
- Prefer `collections.abc` types (`Sequence`, `Mapping`, `Iterable`) over `typing` equivalents.
- Use `TypeAlias` for complex reusable types.
- Run `mypy` or `pyright` on generated code.

---

## Error Handling

### Specific exceptions over bare `except`

```python
try:
    data = json.loads(raw)
except json.JSONDecodeError as exc:
    raise ValueError(f"Invalid JSON in {path!r}: {exc}") from exc
```

- Never use bare `except:` or `except Exception:` unless re-raising.
- Always chain exceptions with `raise ... from exc` to preserve traceback context.
- Use `contextlib.suppress(ExceptionType)` for intentionally ignored exceptions.

### Custom exceptions

```python
class AppError(Exception):
    """Base exception for this application."""

class ConfigError(AppError):
    """Raised when configuration is invalid."""
```

---

## Argument Parsing with argparse

```python
import argparse
import sys

def build_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(
        description="Tool description here.",
        formatter_class=argparse.RawDescriptionHelpFormatter,
        epilog="Examples:\n  %(prog)s input.csv -o out/\n",
    )
    parser.add_argument("input", type=Path, help="Input file path")
    parser.add_argument("-o", "--output", type=Path, default=Path("out"),
                        help="Output directory (default: %(default)s)")
    parser.add_argument("-v", "--verbose", action="store_true")
    parser.add_argument("--dry-run", action="store_true",
                        help="Preview actions without making changes")
    return parser

def main(argv: list[str] | None = None) -> int:
    args = build_parser().parse_args(argv)
    # ...
    return 0

if __name__ == "__main__":
    sys.exit(main())
```

---

## Logging

> The format, levels, and color rules below implement the workspace-wide standard. See `.github/skills/cli-output/SKILL.md` for the canonical spec and ready-to-paste implementation.

```python
import logging
import sys

def configure_logging(verbose: bool = False) -> None:
    level = logging.DEBUG if verbose else logging.INFO
    logging.basicConfig(
        level=level,
        format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
        datefmt="%Y-%m-%dT%H:%M:%S",
        stream=sys.stderr,
    )

log = logging.getLogger(__name__)

# Usage
log.info("Processing %d files", count)
log.debug("Details: %r", details)
log.warning("Skipping unreadable file: %s", path)
log.error("Failed to connect: %s", exc)
```

- Always use `logging` instead of `print()` for status/diagnostic output in libraries and tools.
- Use `__name__` as the logger name, not a hard-coded string.
- Use `%` formatting for lazy evaluation in log calls, not f-strings.

---

## File and Path Handling (pathlib)

Always prefer `pathlib.Path` over `os.path`.

```python
from pathlib import Path

def read_config(path: Path) -> dict:
    if not path.is_file():
        raise FileNotFoundError(f"Config not found: {path}")
    return json.loads(path.read_text(encoding="utf-8"))

# Safe temp files
import tempfile
with tempfile.NamedTemporaryFile(suffix=".json", delete=False) as tmp:
    tmp_path = Path(tmp.name)
try:
    process(tmp_path)
finally:
    tmp_path.unlink(missing_ok=True)
```

---

## Context Managers and Resource Safety

```python
# Always use `with` for file I/O, DB connections, locks, etc.
with Path("data.csv").open(encoding="utf-8") as fh:
    reader = csv.DictReader(fh)
    rows = list(reader)

# Custom context manager
from contextlib import contextmanager

@contextmanager
def managed_connection(dsn: str):
    conn = create_connection(dsn)
    try:
        yield conn
    finally:
        conn.close()
```

---

## Comprehensions and Generators

```python
# Prefer comprehensions over map/filter for clarity
names = [user.name for user in users if user.active]

# Use generators for large sequences to avoid loading everything into memory
total = sum(row["value"] for row in read_large_csv(path))

# Dict / set comprehensions
lookup = {item.id: item for item in items}
unique_tags = {tag for post in posts for tag in post.tags}
```

---

## Common Gotchas

```python
# Mutable default arguments — NEVER do this
def bad(items=[]):       # items is shared across calls!
    items.append(1)

# Correct pattern
def good(items=None):
    if items is None:
        items = []
    items.append(1)

# Late binding in closures
# WRONG: all lambdas capture the same `i`
funcs = [lambda: i for i in range(3)]

# CORRECT: capture by default argument
funcs = [lambda i=i: i for i in range(3)]

# Integer identity is only guaranteed for small ints
# WRONG:  if some_int is 0
# CORRECT: if some_int == 0
```

---

## Virtual Environment and Dependencies

### Setup (uv — recommended)

```bash
# Install uv (fast modern package manager)
pip install uv

# Create venv and install deps
uv venv
uv pip install -r requirements.txt

# Or with pyproject.toml
uv sync
```

### Setup (standard venv + pip)

```bash
python3 -m venv .venv
source .venv/bin/activate          # Linux/macOS
.venv\Scripts\Activate.ps1         # Windows (PowerShell)
pip install -r requirements.txt
```

### Dependency file guidelines

- `requirements.txt` — pinned exact versions for deployment/reproducibility.
- `pyproject.toml` (PEP 517/621) — preferred for packages and projects; specify version ranges.
- Always pin transitive deps with `pip freeze > requirements.lock` or `uv lock`.

---

## Project Structure

```
my_project/
├── src/
│   └── my_package/
│       ├── __init__.py
│       ├── cli.py
│       ├── core.py
│       └── utils.py
├── tests/
│   ├── conftest.py
│   └── test_core.py
├── pyproject.toml
├── README.md
└── .python-version
```

---

## Testing with pytest

```python
# tests/test_core.py
import pytest
from my_package.core import process

def test_process_valid_input():
    result = process({"key": "value"})
    assert result["status"] == "ok"

def test_process_empty_raises():
    with pytest.raises(ValueError, match="Input cannot be empty"):
        process({})

@pytest.mark.parametrize("value,expected", [
    (1, "one"),
    (2, "two"),
    (3, "three"),
])
def test_label(value, expected):
    assert label(value) == expected
```

- Use `pytest` as the default test runner.
- Keep tests in `tests/` directory, mirroring the `src/` structure.
- Use `conftest.py` for shared fixtures.
- Aim for `pytest --tb=short -q` as the default run command.
- Mock external calls with `pytest-mock` or `unittest.mock`.

---

## Subprocess and Shell Calls

```python
import subprocess

# Preferred: capture output with check=True
result = subprocess.run(
    ["git", "log", "--oneline", "-10"],
    capture_output=True,
    text=True,
    check=True,   # raises CalledProcessError on non-zero exit
)
print(result.stdout)

# Avoid shell=True unless absolutely necessary — it introduces injection risks
```

- Never use `os.system()`.
- Always pass commands as a list, not a single string, unless `shell=True` is required.
- Always set `check=True` or explicitly handle non-zero exit codes.

---

## Concurrency Patterns

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

def process_all(paths: list[Path], max_workers: int = 8) -> list[Result]:
    results = []
    with ThreadPoolExecutor(max_workers=max_workers) as pool:
        futures = {pool.submit(process_one, p): p for p in paths}
        for future in as_completed(futures):
            path = futures[future]
            try:
                results.append(future.result())
            except Exception as exc:
                log.warning("Failed to process %s: %s", path, exc)
    return results
```

- `ThreadPoolExecutor` for I/O-bound work.
- `ProcessPoolExecutor` for CPU-bound work.
- `asyncio` for high-concurrency async I/O (prefer when the underlying library is async-native).

---

## Security Checklist

- [ ] No hardcoded credentials, tokens, or API keys — use `os.environ` or a secrets manager.
- [ ] All external input validated before use; avoid `eval()` and `exec()` with untrusted data.
- [ ] `subprocess` calls use list form (not `shell=True`) unless absolutely necessary.
- [ ] File paths from user input normalized with `Path.resolve()` and checked against allowed directories.
- [ ] Dependencies pinned and audited (`pip audit` or `safety check`).
- [ ] Sensitive data not logged or printed.

---

## Behavioral Conventions

- Generated scripts should always have a `main()` guard: `if __name__ == "__main__": sys.exit(main())`.
- Generate docstrings for all public functions, classes, and modules.
- Prefer `pathlib.Path` over `os.path` and `open()` over lower-level file APIs.
- When invoking shell commands, default to `check=True` and capture output.
- Always recommend running `mypy --strict` (or `pyright`) and `ruff check .` before committing generated code.
