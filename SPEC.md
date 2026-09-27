# Query Drill — project spec

A local practice app for writing relational algebra (RA) and SQL by hand, graded automatically. Built for COSC 3337 (Database System Concepts, 7th ed.) and designed to grow unit by unit as the course moves forward.

Owner: Jabari Prewitt (GitHub: lowkeybari)
Status: v0 spec, September 2026

---

## 1. Why this exists

The exam asks students to write RA and SQL on paper. Multiple-choice practice doesn't build that skill. This app gives one question at a time, has the student write **both** the RA and the SQL, and checks each answer by running it against a small database built to expose common mistakes.

Goals:

- Practice writing queries quickly, with less typing (clause tiles and operator tiles).
- Grade objectively by comparing results, not by pattern-matching text.
- Add new material by adding data files, not by changing code.
- Track which topics keep going wrong.

Non-goals for now: multi-user accounts, hosting, mobile app, ER diagram grading (see §10).

---

## 2. Core loop

1. The app shows a question: level, domain, prompt, and the schema with keys underlined.
2. The student writes:
   - **RA** in a text box, with a row of operator tiles (σ Π ⋈ × ∪ ∩ − ρ γ ← ∧ ∨ ¬ ≠ ≤ ≥) that insert at the cursor.
   - **SQL** as clause lines: clicking `+ select`, `+ from`, `+ where`, `+ group by`, `+ having` or `+ order by` adds a labelled line to type into. Lines stay in the order clicked, because clause order is part of what's being tested. Helper chips (distinct, and, or, not, join, using ( ), on, as, is null, like, desc, *) insert into the focused line.
3. Submit. The app grades RA and SQL separately (§5) and shows:
   - pass or fail for each
   - a result diff: expected rows vs. your rows, with missing and extra rows highlighted
   - the question's "check for" notes and the reference answers
4. The student marks the result (or it's recorded automatically) and picks **Next**, **Retry**, or **Next weak question**.

A question can ask for RA only, SQL only, or both. The default is both.

---

## 3. Difficulty ladder

Matches the instructor's Jeopardy review board.

| Level | What it tests |
|---|---|
| 100 | One table: σ/Π, `select-from-where`, `and`/`or`, `like`, `is null`, `order by`, `distinct` |
| 200 | Joins (2–3 tables), renaming and self-joins, set operations (∪ ∩ −, `union`/`intersect`/`except`) |
| 300 | Aggregation: γ, `group by`, `having`, `count(*)` vs `count(col)` vs `count(distinct col)`, "not in X" via − |
| 400 | Multi-step (assignment ←), outer joins, "both X and Y", "never", counting r × s results |
| 500 | Judge: two student solutions; pick the right one and explain why the other fails |

Recurring traps the question bank must cover (from the instructor):

- `where` vs `having` (the filter-before-average bug)
- a single σ vs set difference ("not in Spring" on one tuple removes nothing)
- `= null` vs `is null`
- arity mismatch in ∪ and −
- natural join matching an unintended same-named column (e.g. `deptname`)
- missing connector table in a join path
- direction words: under/below/before → `<`, "at least" → `>=`

---

## 4. Content model (the part that grows)

All course content lives in `content/`. Adding material means adding files there.

```
content/
  datasets/
    library/   schema.sql   seed.sql
    hospital/  schema.sql   seed.sql
    airline/   schema.sql   seed.sql
    hotel/     schema.sql   seed.sql
    registrar/ schema.sql   seed.sql
  questions/
    unit2-relational-algebra.yaml
    unit3-intro-sql.yaml
    unit4-sql-aggregates-joins.yaml
    ...
```

### 4.1 Datasets

- `schema.sql` holds `create table` statements with primary and foreign keys.
- `seed.sql` holds small data (5–20 rows per table) **designed to catch mistakes**. Each dataset must include at least:
  - a null in a column questions filter on
  - a row that has no match in a join (a member with no loans, a room never stayed in)
  - duplicate values that `distinct` should collapse
  - two people with the same name but different keys
  - ties in values that get maximised or sorted
  - where relevant, same-named non-key columns that break a natural join

A seed file should state which trap each row is there for, using SQL comments.

### 4.2 Question format

```yaml
- id: hotel-100-double-under-200
  unit: 3
  level: 100
  domain: hotel
  tags: [selection, projection, comparison]
  prompt: >
    Find the room_no and rate of every room of type 'Double'
    with a rate under 200.
  answer_kinds: [ra, sql]
  compare: set            # set | bag | ordered
  reference_sql: |
    select room_no, rate
    from room
    where type = 'Double' and rate < 200;
  reference_ra: |
    Π room_no, rate (σ type = 'Double' ∧ rate < 200 (room))
  check_for:
    - "Under 200 means < 200."
    - "Row conditions go in where, joined with and. group by is only for 'for each'."
  why: >
    σ ↔ where, Π ↔ select, relation ↔ from, ∧ ↔ and.
```

Judge questions (level 500) use a different shape:

```yaml
- id: hosp-500-fee-having
  level: 500
  type: judge
  domain: hospital
  prompt: >
    For each doctor who has charged at least one fee above 500,
    show their average fee over all of their appointments.
  options:
    A: |
      select doc_id, avg(fee) from appointment
      where fee > 500 group by doc_id;
    B: |
      select doc_id, avg(fee) from appointment
      group by doc_id having max(fee) > 500;
  correct: B
  explanation_required: true
  check_for:
    - "A averages only the fees above 500 (filter-before-average bug)."
```

For Judge questions the app runs both options and shows how their results differ. That result diff is the evidence the student's explanation should point to.

### 4.3 Content validation

`python -m app.validate` must:

- parse every YAML file against the schema
- build every dataset
- run every `reference_sql` and `reference_ra` and confirm the two return the same result
- fail loudly on duplicate ids, unknown domains, or reference answers that return zero rows (usually a sign of bad seed data)

This runs in CI and before every commit.

---

## 5. Grading

### 5.1 SQL

1. Reassemble the clause lines into one statement, in the order the student wrote them. Clauses in the wrong order are a syntax error, and the app says which clause is out of place.
2. Refuse anything that isn't a single read-only `select` (no `;`-chained statements, no DDL or DML). Open the database read-only.
3. Run the student's query and the reference against a fresh in-memory copy of the dataset, with a timeout.
4. Compare using the question's `compare` mode:
   - `set` compares distinct rows (the default)
   - `bag` compares row counts too (use when the question depends on duplicates, e.g. `union all`)
   - `ordered` compares order too (use when the question says "sorted" or "highest first")
5. Compare values only, not column names. Column count must match.
6. On failure, report: a SQL error message in plain words; wrong column count; missing rows; extra rows; wrong order.

### 5.2 Relational algebra

RA can't be run directly, so the app includes a parser and translator: **RA text → AST → SQL**, which is then graded exactly like SQL, with set semantics.

Accepted syntax, Unicode or ASCII:

| Operator | Unicode | ASCII |
|---|---|---|
| select | `σ cond (r)` | `select[cond](r)` or `sigma cond (r)` |
| project | `Π a, b (r)` | `project[a,b](r)` or `pi a, b (r)` |
| natural join | `r ⋈ s` | `r join s` |
| theta join | `r ⋈ cond s` | `r join[cond] s` |
| product | `r × s` | `r x s` |
| union, intersect, minus | `∪ ∩ −` | `union intersect minus` |
| rename | `ρ T (r)`, `ρ T(a,b) (r)` | `rename[T](r)` |
| aggregate | `g γ f(a) as x (r)` | `gamma[g; f(a) as x](r)` |
| assignment | `t ← expr` | `t <- expr` |
| and, or, not | `∧ ∨ ¬` | `and or not` (also `^`) |
| comparisons | `= ≠ < ≤ > ≥` | `= != <> < <= > >=` |

Rules:

- Multi-line input where each line is an assignment, with the last line as the result.
- Strings in single quotes. The error message for double quotes should explain why.
- Parser errors point to the position and name the problem (e.g. "unbalanced parentheses", "σ needs a condition before the relation").
- Check union compatibility before translating, and report arity mismatches as an RA error rather than a SQLite error.
- Π always translates to `select distinct`.

Translation targets (SQLite 3.39+ for `full outer join`):

- σ → `where`
- Π → `select distinct`
- ⋈ natural → `natural join`
- × → cross join
- ∪ ∩ − → `union` / `intersect` / `except`
- ρ → subquery alias
- γ → `group by` with aggregates
- ← → CTE (`with t as (...)`)

### 5.3 Optional AI explanations (later)

A "Why is this wrong?" button sends the question, the student's answer, the reference answer and the result diff to the Claude API, and shows a short explanation. The grade itself always comes from result comparison, never from the model. The API key is read from an environment variable and is never committed.

---

## 6. Progress tracking

A separate `progress.db` (SQLite, git-ignored):

- `attempt(id, question_id, kind, submitted_text, passed, error_type, created_at)`
- `question_status(question_id, kind, status)` where status is new, missed, partly or passed

UI:

- The ladder view shows each question as a tile coloured by status.
- **Next weak question** chooses from missed questions first, then untried ones at the lowest level that isn't passed yet.
- **Weak topics** view: pass rate by tag (joins, having, nulls, set-difference, ...).

---

## 7. Tech stack

- Python 3.12, FastAPI, Uvicorn
- SQLite through the standard library `sqlite3`
- PyYAML and Pydantic for loading and validating content
- RA parser: Lark (or a hand-written recursive-descent parser if that's more instructive)
- Frontend: plain HTML, CSS and JavaScript served by FastAPI, no build step
- Tests: pytest
- Lint and format: ruff

### Project layout

```
query-drill/
  CLAUDE.md
  SPEC.md
  README.md
  pyproject.toml
  app/
    main.py            FastAPI app and routes
    content.py         load and validate YAML and datasets
    sql_grader.py      run and compare SQL
    ra/
      grammar.lark
      parser.py        RA text -> AST
      to_sql.py        AST -> SQL
    progress.py
    validate.py        python -m app.validate
  static/
    index.html
    app.js
    style.css
  content/
    datasets/...
    questions/...
  tests/
    test_sql_grader.py
    test_ra_parser.py
    test_ra_to_sql.py
    test_content.py
```

---

## 8. Build order

Each step should be small enough for one Claude Code session, end with passing tests, and end with a commit.

1. **Skeleton:** repo, `pyproject.toml`, FastAPI hello world, pytest, ruff, `.gitignore`.
2. **Content loading:** the Pydantic question model, YAML loader, and dataset builder. Add the hotel dataset and 3 level-100 questions.
3. **SQL grader:** run and compare in set, bag and ordered modes, the read-only guard, and the timeout. Write tests against trap rows.
4. **Minimal UI:** show a question, the SQL clause tiles, submit, and a result diff.
5. **Validation command:** `python -m app.validate` and a pre-commit hook.
6. **Progress tracking:** attempts, statuses, the ladder view, and Next weak question.
7. **RA parser, part 1:** σ, Π, relation names, conditions, parentheses, plus the RA tile input in the UI.
8. **RA parser, part 2:** ⋈, ×, ∪, ∩, −, ρ, with a union-compatibility check.
9. **RA parser, part 3:** γ and ← assignment.
10. **Judge questions:** a type with two options, a result diff between them, and a free-text explanation box.
11. **Fill the question bank:** all five domains, levels 100–500.
12. **Optional:** AI explanations (§5.3).

---

## 9. Course coverage and how to extend

| Unit | Material | In the app |
|---|---|---|
| 1 | Intro and design (Ch. 1) | conceptual, not query-based, so not included |
| 2 | Relational model and RA (Ch. 2) | RA questions, levels 100–400 |
| 3 | Intro SQL (Ch. 3) | SQL questions, levels 100–300 |
| 4 | SQL 2: aggregates and joins (Ch. 3–4) | SQL questions, levels 200–500 |
| 5 | ER models (Ch. 6) | not yet (see §10) |
| later | nested subqueries, views, etc. | add when covered in lecture |

To add a new topic:

1. Add or extend a dataset if the topic needs new trap rows.
2. Add a YAML file under `content/questions/`.
3. Run `python -m app.validate`.
4. Only change code if the topic needs a new grading capability (e.g. subqueries in RA, or `with` in SQL). If it does, write that change as its own step, with tests.

Only include material that has been covered in lecture. The course is at Lecture 5.1 as of this spec, so there are no nested subqueries and no division operator yet.

---

## 10. Future ideas

- ER exercises: a scenario → identify entities, attributes (simple/composite, single/multivalued, derived), cardinality, participation and weak entity sets. Grade with structured fields rather than a drawing.
- Timed "exam mode" with mixed levels.
- Export missed questions to a printable review sheet.
- Deploy to the DigitalOcean droplet for use from a phone.
