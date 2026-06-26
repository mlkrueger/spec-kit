# Profile: Python (pytest)

Default test stack for Python services and libraries. Adapt if the repo clearly uses a different runner
(`unittest`, `nose2`) — but prefer pytest when nothing is established. Manage the environment with
whatever the repo uses (`uv` preferred, else Poetry, pip-tools, Hatch).

## Runner & commands

- **Runner:** `pytest`.
- **Whole suite:** `pytest`
- **One file:** `pytest tests/unit/test_foo.py`
- **One test:** `pytest tests/unit/test_foo.py::test_name` (or `-k "expr"`)
- **Coverage:** `pytest --cov=<pkg> --cov-report=term-missing` (via `pytest-cov`)

## Layers (which the build/acceptance agents target)

- **Unit** (build plan, inline `tddCases`) — pure functions, domain models (dataclasses), validation,
  parsing, date/recurrence math. Aim for near-exhaustive branch coverage here.
- **Integration** (build plan, inline `tddCases`) — request handlers / service functions with the
  datastore and third-party clients mocked at one seam. For web frameworks use the app's test client
  (FastAPI `TestClient`, Django test client, Flask `test_client`).
- **Component** — usually N/A for pure-Python backends.
- **End-to-end** (acceptance plan) — for a service: black-box HTTP against a built app with a mocked
  backend; for a CLI: invoke the entry point and assert on stdout/exit code (`subprocess` or the
  framework's runner, e.g. Click/Typer `CliRunner`).

## File naming & location

- Tests in `tests/`, mirroring the package layout; files named `test_*.py`, functions `test_*`.
- Separate `tests/unit/`, `tests/integration/`, `tests/e2e/` so CI targets them independently — this is
  the **naming/location infix** a build ticket's `modulesInScope` test path and an acceptance ticket's
  E2E spec path must use.
- Shared fixtures in `conftest.py` at the appropriate scope.

## Mocking seam & determinism

- Inject dependencies (clients, clock) via constructor/params; prefer hand-rolled fakes over
  `unittest.mock` auto-mocks when asserting call *arguments*.
- **Inject `now`** — never call `datetime.now()` inside logic under test. Pin a fixed `tz` in CI
  (`TZ=UTC` and/or `freezegun`/`time-machine` for clock control) and add DST/boundary cases.
- Reset shared state in fixtures with `yield` teardown; tests must pass in any order.

## Gotchas to encode as regression tests

- **Naive vs aware datetimes** — mixing `datetime.now()` (naive) with aware UTC values; serialization
  round-trips dropping tzinfo.
- **Mutable default arguments** (`def f(x=[])`) — shared state across calls.
- **Float/Decimal money math** — assert with `Decimal`, not `float` equality.
- **Async** — `pytest-asyncio` event-loop scope; un-awaited coroutines silently passing; cancellation
  paths.
- **Pydantic vs dataclass boundaries** — validation happens at the Pydantic edge; assert invalid input
  is rejected *there*, not deep in the logic. (Per cross-project style: dataclasses for domain models,
  Pydantic for API schemas — don't mix.)
- **Exception paths** — a dependency raising must be handled, and fail-closed for authz; test the error
  branch explicitly.
