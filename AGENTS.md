# AGENTS.md

## Project

Python 3.12 application.

## Development

Run tests with:

    pytest

Run linting with:

    ruff check .

Run formatting with:

    ruff format .

Run type checking with:

    mypy src

## Code style

- Use Python 3.12 syntax.
- Add type annotations to public functions.
- Prefer pathlib over os.path.
- Keep functions small.
- Do not modify tests merely to make failing code pass.
- Add tests for new behavior.

## Before completing a task

Run:

    ruff check .
    pytest
