# Repository Guidelines

## Project Structure & Module Organization

The repo targets uv-managed Python 3.13. Source lives in `src/garth_mcp_server`
with the `FastMCP` server definitions and tool wrappers, while `main.py` is a
thin entry point used by `uvx` for local smoke tests. Dependency metadata stays
in `pyproject.toml`; the `Makefile` drives automation; generated artifacts (for
example `build/`, `dist/`, `__pycache__`) should remain untracked. When you add
tests, keep them in `tests/` mirroring the package path and name files
`test_<module>.py`.

## Build, Test, and Development Commands

Run `make install` once to install the project in editable mode plus dev/lint
extras and to configure pre-commit hooks. Use `make format` to enforce Ruff
formatting and auto-fix low-risk lint issues. `make lint` runs Ruff format
checks, Ruff linting, and `mypy` for type safety, while `make codespell`
catches spelling regressions. `make all` is the CI-equivalent target (lint +
codespell). For manual loops, `uv run garth_mcp_server` exposes the MCP server
locally, and `uv run pytest` executes any pytest suites you add.

## Coding Style & Naming Conventions

Stick to PEP 8 with 4-space indentation, exhaustive type hints, and
module-level docstrings describing intent. Favor small, pure functions over new
classes unless object lifecycles genuinely improve clarity. Keep comments
focused on “why” (workarounds, protocol quirks) rather than restating code. Let
Ruff own formatting and linting; never mix `pip` into workflows—rely on `uv`
and extras such as `[dev,linting]`.

## Testing Guidelines

Target pytest with files named `test_*.py` and shared fixtures in
`conftest.py`. Provide regression tests for every new tool endpoint or bugfix;
cover both successful API responses and GARTH_TOKEN failure messages. If you
introduce UI or interactive layers, accompany the change with Playwright MCP
end-to-end coverage to exercise the user-facing flow before opening a pull
request.

## Commit & Pull Request Guidelines

Commits follow the existing concise, imperative style (`Fix activities return
type hint (#14)`), optionally appending the tracking issue in parentheses.
Rebase before pushing to keep history linear. Open pull requests with `gh pr
create` and include: intent summary, bullet list of notable code changes/tests,
any GARTH_TOKEN or `.env` considerations, screenshots or logs for UI flows, and
identified risks. Request review only after `make all` passes locally to keep CI
green.

## Security & Configuration Tips

Authenticate via `uvx garth login` and export the resulting token as
`GARTH_TOKEN`; load it from `.env` with `python-dotenv` instead of hardcoding.
Guard API credentials and personal health data—sanitize logs before sharing.
Clean caches with `make clean` before publishing release artifacts to avoid
leaking local paths.
