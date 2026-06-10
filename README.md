# Agent Tool Endpoint

Minimal FastAPI backend for running registered tools and storing traces in SQLite.

## Setup

```bash
pip install -e ".[dev]"
```

## Run

```bash
uvicorn app.main:app --reload
```

Server listens on `http://127.0.0.1:8000`. Interactive docs at `/docs`.

## Test

```bash
pytest -q
```

## API

### POST /agent/run

Run a registered tool with a message.

```bash
curl -X POST http://127.0.0.1:8000/agent/run \
  -H "Content-Type: application/json" \
  -d '{"message": "12*8", "tool": "calculator"}'
```

Success (200):
```json
{"trace_id": "...", "tool": "calculator", "result": 96.0, "latency_ms": 3}
```

Failure envelope (4xx/5xx):
```json
{"trace_id": "...", "error": {"code": "TOOL_NOT_FOUND", "message": "tool 'weather' is not registered"}}
```

| 场景 | HTTP | error.code |
|---|---|---|
| 参数缺失/不合规 | 422 | `INVALID_REQUEST` |
| tool 未注册 | 404 | `TOOL_NOT_FOUND` |
| 工具执行抛业务异常 | 400 | `TOOL_EXECUTION_ERROR` |
| 未预期异常 | 500 | `INTERNAL_ERROR` |

### GET /agent/traces/{trace_id}

Fetch a recorded trace by id.

```bash
curl http://127.0.0.1:8000/agent/traces/<trace_id>
```

Success (200) includes `input`, `result` (on success) or `error` (on failure), `latency_ms`, `created_at`. Not found returns 404 `TRACE_NOT_FOUND`.

## Built-in Tools

- `echo` — returns the input message
- `calculator` — evaluates `+ - * /` expressions with parentheses and unary minus (recursive-descent parser, no `eval`)

## Adding a New Tool

1. Create `app/tools/<name>.py`:

```python
from app.registry import register_tool

@register_tool(name="weather", description="Lookup weather")
def run(message: str) -> str:
    return f"sunny in {message}"
```

2. Add one import line to `app/tools/__init__.py`:

```python
from . import calculator, echo, weather
```

No changes to `main.py` / `dispatcher.py` / `trace_store.py` are needed.

## Configuration

- `TRACE_DB_PATH` (env): SQLite file location, default `./traces.db`
