# rho

A local FastAPI + SQLite app for practicing relational algebra and SQL. Answers are graded by running them against small datasets and comparing results. Read `SPEC.md` before starting any task. It is the source of truth for features, formats and build order.

## Who I am and how I work

- I'm a CS student building this for my database course (COSC 3337) and as a portfolio project. I need to understand and be able to explain every part of it.
- Work in small steps that match `SPEC.md` §8. One step per session unless I say otherwise.
- Before writing code for a step, give a short plan: files you'll touch and tests you'll add. Wait for my OK.
- After a change, explain what you did and why in plain language, including any design choice I should know about.
- Don't add features, dependencies or files outside the current step without asking.

## Commands

- Install: `pip install -e ".[dev]"`
- Run: `uvicorn app.main:app --reload`
- Test: `pytest`
- Lint and format: `ruff check . && ruff format .`
- Validate content: `python -m app.validate`

## Conventions

- Python 3.12, type hints everywhere, Pydantic models for anything loaded from files.
- Course content lives only in `content/` (YAML questions, SQL datasets). Never hard-code questions in Python or JavaScript.
- Frontend is plain HTML, CSS and JavaScript in `static/`. No frameworks or build step.
- Every grader or parser change comes with pytest tests. Include tests that use the trap rows in the seed data.
- SQL strings in content use single quotes for string literals.

## Rules that must not be broken

- Never change a reference answer or seed data just to make a test pass. If a reference looks wrong, stop and tell me.
- Grading always comes from result comparison. An AI explanation, if added, never decides pass or fail.
- The student's SQL runs read-only, one `select` statement, with a timeout.
- Only add course material that has been covered in lecture (see `SPEC.md` §9).
- Never commit secrets. The API key, if used, comes from an environment variable.

## Git

- Commit after each working step with a clear message: what changed and why.
- Run `pytest` and `python -m app.validate` before committing.
