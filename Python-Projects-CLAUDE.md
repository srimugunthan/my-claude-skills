# CLAUDE.md — Python Project Guidelines

This file defines conventions, constraints, and expectations for Claude when working in this Python codebase.

---

## Project Overview

> _Fill in: What does this project do? Who uses it? What is its core domain?_

Example:
```
This is a fraud detection microservice built with FastAPI. It exposes REST endpoints
for real-time scoring and batch inference. The primary consumers are internal risk teams.
```

---

## Python Version & Runtime

- **Target Python version:** 3.11+
- **Runtime environment:** _e.g., Docker, Lambda, bare metal, Jupyter_
- **Package manager:** _pip / poetry / uv / conda_ — specify which one
- **Virtual environment:** Always assume a venv is active. Do not suggest global installs.

---

## Project Structure

```
project-root/
├── src/
│   └── mypackage/         # Main source package
│       ├── __init__.py
│       ├── models/        # Pydantic / SQLAlchemy / dataclass models
│       ├── services/      # Business logic layer
│       ├── api/           # FastAPI routers or Flask blueprints
│       └── utils/         # Shared utilities
├── tests/                 # All tests go here, mirror src/ structure
├── notebooks/             # Exploratory notebooks only, not production code
├── scripts/               # One-off CLI scripts
├── pyproject.toml         # Project metadata and dependencies
├── .env.example           # Template for environment variables
└── CLAUDE.md              # This file
```

Claude must respect this structure. Do not suggest flat layouts or put logic directly in `__init__.py`.

---

## Code Style & Formatting

- **Formatter:** `black` (line length: 88) — do not suggest alternatives
- **Linter:** `ruff` — preferred over flake8/pylint
- **Type checking:** `mypy` with strict mode enabled
- **Import ordering:** `isort` compatible (ruff handles this)
- All new code must be **fully type-annotated**
- Avoid `Any` unless absolutely necessary; justify with a comment if used

```python
# Good
def compute_risk_score(transaction: Transaction, threshold: float = 0.5) -> RiskResult:
    ...

# Bad — missing types
def compute_risk_score(transaction, threshold=0.5):
    ...
```

---

## Naming Conventions

| Element | Convention | Example |
|---|---|---|
| Module | `snake_case` | `fraud_detector.py` |
| Class | `PascalCase` | `TransactionScorer` |
| Function / method | `snake_case` | `compute_risk_score()` |
| Constant | `UPPER_SNAKE_CASE` | `MAX_RETRY_COUNT` |
| Private | leading underscore | `_internal_helper()` |
| Type alias | `PascalCase` | `FeatureMatrix = np.ndarray` |

---

## Dependency Rules

- Add dependencies to `pyproject.toml`, not ad-hoc `pip install` commands
- **Do not introduce new third-party libraries** without flagging it explicitly in your response
- Prefer stdlib over third-party for simple utilities (e.g., use `pathlib` not `os.path`)
- Approved ML/data stack: `numpy`, `pandas`, `scikit-learn`, `torch`, `pydantic`, `sqlalchemy`
- HTTP clients: use `httpx` (async-first), not `requests`
- Do not use `pickle` for serialization in production paths — use `joblib` or `safetensors`

---

## Error Handling

- Always use **explicit exception types** — never bare `except:`
- Raise domain-specific exceptions defined in `src/mypackage/exceptions.py`
- Use `logging` (not `print`) for all diagnostic output
- Log at the right level: `DEBUG` for dev traces, `INFO` for key events, `ERROR` for failures
- **Never catch an error without logging it.** Always propagate errors up or raise explicitly. Silent failures — swallowed exceptions, bare `pass`, returning `None` without indication — are forbidden.
- **Never return `None` or `undefined` silently to signal failure.** Raise an explicit exception with a descriptive message instead.

```python
# Good
try:
    result = score_transaction(txn)
except ValidationError as e:
    logger.error("Invalid transaction payload: %s", e)
    raise TransactionScoringError("Validation failed") from e

# Bad — silent failure
try:
    result = score_transaction(txn)
except Exception:
    pass  # never do this

# Bad — silent None return
def score_transaction(txn):
    try:
        ...
    except Exception:
        return None  # caller has no idea something went wrong
```

### Logging at Service Boundaries

- Add `logger.debug()` at the entry and exit of every **service-layer function and async flow** — log inputs on entry, outputs (or error) on exit.
- This is mandatory for async functions and any function that calls external I/O (DB, HTTP, queue).
- Internal utility functions and simple transformations are exempt — use judgment.

```python
async def fetch_merchant_profile(merchant_id: str) -> MerchantProfile:
    logger.debug("fetch_merchant_profile | start | merchant_id=%s", merchant_id)
    try:
        profile = await db.get_merchant(merchant_id)
        logger.debug("fetch_merchant_profile | done | result=%s", profile)
        return profile
    except DBTimeoutError as e:
        logger.error("fetch_merchant_profile | timeout | merchant_id=%s", merchant_id)
        raise MerchantFetchError(f"Timeout fetching {merchant_id}") from e
```

---

## Testing

- **Framework:** `pytest`
- **Coverage target:** 80% minimum on `src/`
- Every new function or class must have at least one corresponding test
- Test file naming: `test_<module_name>.py` in `tests/` mirroring `src/`
- Use `pytest.fixture` for shared setup; avoid `setUp`/`tearDown` patterns
- Mock external I/O (DB, HTTP, filesystem) — tests must be runnable offline
- Use `pytest-asyncio` for async code

```python
# Good
def test_risk_score_returns_high_for_flagged_merchant(mock_db):
    txn = Transaction(merchant_id="FLAGGED_001", amount=9999.0)
    result = compute_risk_score(txn)
    assert result.score > 0.8
```

---

## Async Guidelines

- Use `async def` and `await` consistently — do not mix sync blocking calls inside async functions
- Use `asyncio.gather()` for concurrent I/O, not sequential `await` in a loop
- Use `httpx.AsyncClient` as a context manager
- Avoid `asyncio.run()` inside library code — only in entrypoints

---

## Configuration & Secrets

- All config comes from environment variables — use `pydantic-settings` with a `Settings` class
- Never hardcode secrets, API keys, or connection strings
- `.env` files are for local dev only — never committed to version control

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    db_url: str
    model_threshold: float = 0.5

    class Config:
        env_file = ".env"
```

---

## Documentation

- All public functions and classes must have **docstrings** (Google style preferred)
- Module-level docstrings are required for non-trivial modules
- Inline comments should explain *why*, not *what*

```python
def compute_risk_score(transaction: Transaction) -> RiskResult:
    """Compute a fraud risk score for the given transaction.

    Args:
        transaction: A validated Transaction object.

    Returns:
        RiskResult with a float score in [0, 1] and a label.
    """
```

---

## What Claude Should NOT Do

- Do not rewrite working code without being asked
- Do not change function signatures silently
- Do not introduce `__all__` exports unless the module is a public API surface
- Do not use `global` variables
- Do not suggest Jupyter notebooks for production logic
- Do not output `requirements.txt` unless explicitly asked — use `pyproject.toml`
- Do not silently swallow exceptions, use `pass` in `except` blocks, or return `None` to signal failure — always raise explicitly
- Do not omit boundary logging in service-layer or async functions

---

## Preferred Patterns

**Prefer dataclasses or Pydantic over raw dicts for structured data:**
```python
from pydantic import BaseModel

class Transaction(BaseModel):
    txn_id: str
    amount: float
    merchant_id: str
```

**Prefer `pathlib.Path` over `os.path`:**
```python
from pathlib import Path
config_path = Path("config") / "settings.yaml"
```

**Prefer generator expressions over list comprehensions when consuming once:**
```python
total = sum(txn.amount for txn in transactions if txn.flagged)
```

---

## Running the Project

```bash
# Install dependencies
poetry install   # or: pip install -e ".[dev]"

# Run tests
pytest tests/ -v --cov=src

# Type check
mypy src/

# Lint + format
ruff check src/ tests/
black src/ tests/
```

---

_Last updated: April 2026_
