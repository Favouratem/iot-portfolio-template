# Module 1

Task 1

Module description

## Learning Goals

> **How to use the learning goals.** Copy the module's learning goals from the course workbook into the block below. Keep the `LG1`, `LG2`, … IDs and a one-word keyword. In each task, write `Proves: LG2` (or link to `[LG2](#lg2)`) so you can find the evidence quickly at checkpoint time. In the **Evidence** line under each goal, link to the task that proves it.

<a id="lg1"></a>
### LG1 — `<CBL/PBL>`


<Explain how this class works — the CBL/PBL philosophy, the portfolio-based project work assessment, and the "moving bar" of a first-time course>

**Evidence:** [Task Inspirations](#task-inspirations)

<a id="lg2"></a>
### LG2 — `<GitHub portfolio>`

<Set up your personal GitHub portfolio from the course template, with a clean first commit and a .gitignore that keeps databases and secrets out of Git.>

**Evidence:** [Task Personal Profile](#task-personal-profile)

<a id="lg3"></a>
### LG3 — `<toolchain>`

<Set up the entire toolchain reproducibly with uv — one fixed Python plus SQLite, DuckDB, Datasette, and more — on Windows, macOS, or Linux, and verify it.>

**Evidence:** [Task What is IoT](#task-what-is-iot)

<a id="lg4"></a>
### LG4 — `<embedded database and a client/server database>`

<Explain — with a measurement, not a slide — the difference between an embedded database and a client/server database, and why most local apps should not run a server.>

**Evidence:** [Task What is IoT](#task-what-is-iot)

<a id="lg5"></a>
### LG5 — `<real rows>`

<Load real rows into SQLite and query them from Python using bound parameters.>

**Evidence:** [Task What is IoT](#task-what-is-iot)
<a id="task-inspirations"></a>

<a id="lg6"></a>
### LG6 — `<LLM-generated query>`

<Take a first LLM-generated query, run it, and begin the habit that defines this course: verify before you trust.>

**Evidence:** [Task What is IoT](#task-what-is-iot)

## Task Inspirations

**Proves:** [LG1](#lg1)

- Inspiration 1
- Inspiration 2

<a id="task-personal-profile"></a>
## Task Personal Profile

**Proves:** [LG2](#lg2)

<a id="task-what-is-iot"></a>
## Task What is IoT

**Proves:** [LG3](#lg3)

Domains and included areas:

-
-

Commonly used (data) protocols:

-
-

Typical devices (appliance or micro controller):

-
-

Challenges:

-
-

## Working with the telemetry database

From the repository root, generate the sample CSV and load it into
`Module01/telemetry.db`:

```powershell
python Module01\generate_telemetry.py
python Module01\load_telemetry.py
```

Open the database explicitly so SQLite does not create a different, empty
database in the current directory:

```powershell
sqlite3 Module01\telemetry.db
```

At the `sqlite>` prompt, these SQLite dot-commands inspect the database; the
table is named `readings`:

```sql
.tables
.schema readings
.timer on
SELECT machine, COUNT(*) AS n FROM readings GROUP BY machine;
EXPLAIN QUERY PLAN SELECT * FROM readings WHERE machine = 'cnc-01';
```

The same checks can be run as one-shot commands:

```powershell
sqlite3 Module01\telemetry.db ".tables"
sqlite3 Module01\telemetry.db ".schema readings"
sqlite3 -cmd ".timer on" Module01\telemetry.db "SELECT machine, COUNT(*) AS n FROM readings GROUP BY machine;"
sqlite3 Module01\telemetry.db "EXPLAIN QUERY PLAN SELECT * FROM readings WHERE machine = 'cnc-01';"
```

Dot-commands such as `.timer` are SQLite shell settings, not SQL statements.
To execute a SQL script file from PowerShell, pipe it to SQLite:

```powershell
Get-Content script.sql | sqlite3 Module01\telemetry.db
```

Other ways to inspect or import the CSV:

```powershell
uv run datasette Module01\telemetry.db --open
uv run sqlite-utils memory Module01\telemetry.csv "SELECT count(*) FROM data"
```

To import the CSV directly, use a separate database so you do not add duplicate
rows to the database populated by `load_telemetry.py`. With the SQLite CLI,
create the table first, then import:

```powershell
sqlite3 telemetry-import.db "CREATE TABLE readings (ts TEXT, machine TEXT, temp_c REAL, vib_rms REAL, spindle_load REAL);"
sqlite3 -cmd ".mode csv" -cmd ".import --skip 1 Module01\telemetry.csv readings" telemetry-import.db
```

Alternatively, use `sqlite-utils` to create and populate a separate database:

```powershell
uv run sqlite-utils insert telemetry-utils.db readings Module01\telemetry.csv --csv
```

## Reflection 1

[Reflection 1](../Reflections/ref01.md)
