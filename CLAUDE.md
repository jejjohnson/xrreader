# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

`xrreader`: authenticated archive readers for xarray geodata. It turns a typed
`Request` (variables + `BBox` + `TimeRange` + vertical) into a service-specific
payload and `download()`s or `open()`s an `xarray.Dataset` from external
geoscience archives — Copernicus Marine (CMEMS), the Climate Data Store (CDS),
and AEMET OpenData. It is the data-acquisition layer feeding `xrtoolz` operators
and `geopatcher` tiling, complementary to `geocatalog` (which indexes files/STAC
rather than fetching arrays). Built with Python 3.12+, uv, pytest, and MkDocs.
The full design is in `docs/design/architecture.md`.

## Common Commands

```bash
make install              # Install all deps (uv sync --all-groups) + pre-commit hooks
make test                 # Run tests: uv run pytest -v
make format               # Auto-fix: ruff format . && ruff check --fix .
make lint                 # Lint code: ruff check .
make typecheck            # Type check: ty check src/xrreader
make precommit            # Run pre-commit on all files
make docs-serve           # Local docs server
```

### Running a single test

```bash
uv run pytest tests/test_example.py::TestClass::test_method -v
```

### Pre-commit checklist (all four must pass)

```bash
uv run pytest -v                              # Tests
uv run --group lint ruff check .              # Lint — ENTIRE repo, not just src/xrreader/
uv run --group lint ruff format --check .     # Format — ENTIRE repo
uv run --group typecheck ty check src/xrreader  # Typecheck — package only
```

**Critical**: Always lint/format with `.` (repo root), not `src/xrreader/`. CI runs `ruff check .` which includes `tests/` and `scripts/`.

## Architecture

### Package structure

All implementation lives in `src/xrreader/`. The public API is re-exported through `src/xrreader/__init__.py`.

### Key directories

| Path | Purpose |
|------|---------|
| `src/xrreader/` | Main package source code |
| `tests/` | Test suite |
| `docs/` | Documentation (MkDocs) |
| `notebooks/` | Jupyter notebooks |
| `scripts/` | Example scripts |

## Documentation Examples

Example notebooks live in `docs/notebooks/` as jupytext percent-format `.py` files. The workflow:

1. Write the `.py` source (jupytext percent format)
2. Convert and execute: `jupytext --to notebook foo.py` then `jupyter nbconvert --execute --inplace foo.ipynb`
3. Delete the `.py` — the executed `.ipynb` is the committed source of truth
4. `mkdocs-jupyter` renders the pre-executed `.ipynb` with `execute: false`

Figures render inline via `plt.show()` — do **not** use `savefig` or commit separate PNG files. The `.ipynb` cell outputs are the single source of rendered figures.

See `.github/instructions/docs-examples.instructions.md` for full standards.

## Coding Conventions

- Google-style docstrings
- `dataclasses` or `attrs` for data containers
- Type hints on all public functions and methods
- Pure functions where possible; side effects isolated and explicit
- Surgical changes only — don't refactor adjacent code or add docstrings to unchanged code

## Plans

Plans and design documents go in `.plans/` (gitignored, never committed). Track work via GitHub issues instead.

## PR Review Comments

When addressing PR review comments, always resolve each review thread after fixing it via the GitHub GraphQL API (`resolveReviewThread` mutation). Do not leave addressed comments unresolved. To obtain the required `threadId`, first list the pull request's review threads via the GitHub GraphQL API (see the "Pull Request Review Comments" section in `AGENTS.md` for a minimal query and end-to-end workflow).

## Code Review

Follow the guidance in `/CODE_REVIEW.md` for all code review tasks.
