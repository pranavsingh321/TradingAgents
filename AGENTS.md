# AGENTS.md

Python ≥3.11, single package (`pip install -e ".[dev]"`). Two top-level dirs ship together: `tradingagents/` (library) and `cli/` (Typer app).

## Commands

```bash
ruff check .          # lint — the only lint CI runs
pytest -q             # tests; integration tests excluded by default addopts
pytest tests/test_foo.py::test_name   # single test
```

- There is no formatter step and no typechecker. `ruff format` is deliberately **not** applied repo-wide (see comment in `pyproject.toml`) — don't run it; whole-repo formatting is deferred to avoid mass PR conflicts. E501 is ignored; line length 100.
- Required order before finishing: `ruff check .` → `pytest -q` (they are separate CI jobs; repo must be fully clean under the strict `select`).
- CI runs pytest on py3.11–3.14 **plus one job with `TZ=America/New_York`** — never write a test that only passes in UTC.

## Testing quirks (read before adding a test)

- `tests/conftest.py` blanks every `TRADINGAGENTS_*` env var and `.env` value **before the package is imported**, and points `results_dir` / `cache` / `memory_log_path` / yfinance tz-cache at temp dirs. Don't rely on your own `.env` or `~/.tradingagents` in tests.
- Network is refused at the socket level (including `curl_cffi`) for every test not marked `@pytest.mark.integration`. Unit tests must mock vendor calls in `tradingagents/dataflows/` — a test that needs a live service must carry the `integration` marker and won't run by default (`-m "not integration"`).
- `--strict-markers`: only `unit`, `integration`, `smoke` are registered. Mark new tests.
- `tests/test_suite_isolation.py` enforces the rules above; tests must not touch the network or the user's home.
- Integration tests are the only ones allowed to be slow/live; the default suite is expected to be fast and hermetic.

## Install / dependency gotchas

- Dependencies live only in `pyproject.toml`; `requirements.txt` is a pointer comment. A new runtime import must be added to `dependencies`, or the CI `smoke-install` job (`pip install .` then `import tradingagents, cli.main`) fails.
- Optional extras: `.[dev]` (ruff, pytest), `.[bedrock]` (AWS Bedrock).
- `.env` is loaded **at package import** from the working directory (`find_dotenv(usecwd=True)`), then `.env.enterprise` (Azure) with `override=False`. Exported shell vars always win.

## Architecture

- Entrypoint: `TradingAgentsGraph.propagate(ticker, date, portfolio=...)` in `tradingagents/graph/trading_graph.py`; save reports with `ta.save_reports(state, ticker)`. `main.py` is the minimal example.
- CLI: `cli/main.py:app` → console script `tradingagents`; also `python -m cli.main`. Headless runs: flags answer per-run questions, `TRADINGAGENTS_*` vars answer config questions; a missing answer aborts before the run starts and names the flag/var.
- Config: `tradingagents/default_config.py` is the source of truth. `_ENV_OVERRIDES` in that file is the **only** place to add a new `TRADINGAGENTS_*` → config-key mapping (type of the default drives coercion). Two model tiers (`quick_think_*` / `deep_think_*`), each with its own provider, model, and backend URL.
- Data access goes through `tradingagents/dataflows/router.py` (vendor chain per data kind: Yahoo, SEC EDGAR point-in-time, FRED, social). LLM providers are in `tradingagents/llm_clients/` (`factory.py`, `capabilities.py`).
- State that persists across runs: memory log `~/.tradingagents/memory/trading_memory.md` (always on; settled after holding periods elapse) and opt-in LangGraph SQLite checkpoints (`--checkpoint`).
- Generated/output artifacts: `results/` is excluded from lint; don't commit it.

## Repo-specific conventions

- **Report labels stay literal English.** `**Rating**:` and `FINAL TRANSACTION PROPOSAL` are parsed by downstream readers; translating or rewording them breaks rating extraction even when `output_language` is non-English (`tests/test_i18n_coverage.py`, `test_rating_integrity.py`).
- Agent briefs are `system_message = "..."` plain strings (a trailing comma makes it a tuple and the model receives the repr), and analyst modules must not contain `FINAL TRANSACTION PROPOSAL` (nothing consumes it). Enforced by `tests/test_prompt_integrity.py`.
- Version is single-sourced in `tradingagents/__version__` (read dynamically by setuptools) — bump it there, not in `pyproject.toml`. New files under `cli/static/` and `tradingagents/assets/*.svg` are picked up by existing package-data globs.
- Lint config notes: `combine-as-imports = true`; `F401` ignored in `__init__.py` (intentional re-exports).
- Releases append to `CHANGELOG.md` (contributor credit per release) and attach built wheel/sdist via `.github/workflows/release-assets.yml`.
