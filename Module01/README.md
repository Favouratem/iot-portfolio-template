# Module 1
Story Question;
Wittkamp Zerspanung GmbH is a small machine shop in Gütersloh that wants to monitor its CNC milling machines. The vendor Cirrus Cloud Systems proposes the classic setup: a cloud server, a PostgreSQL instance, a REST backend, a React frontend, and a monthly invoice. The plant manager, Kai Lehmann — a HSBI Industrial Engineering alumna — asks one question the vendor cannot answer cheaply: "How many rows per day, and who else is reading this?"
# answer #
Kai Lehmann's question challenges whether the proposed cloud architecture is actually justified for a small machine shop.
[ Who else is reading this?] -addresses data ownership and privacy. If the data is sent to a cloud service, it may be accessed by the vendor's infrastructure, administrators, backup systems, monitoring tools, or third-party services involved in running the platform. The plant manager wants to know why the data needs to leave the factory at all if the primary users are local engineers and machine operators.

Looking at the telemetry example in the module, each CNC machine records one reading every 3 seconds. With three machines, that produces 86,400 rows per day (28,800 readings per machine per day × 3 machines). This is a relatively small amount of data that can easily be handled by an embedded database such as SQLite on a local PC or industrial gateway
Module description
1) Explain how this class works — the CBL/PBL philosophy, the portfolio-based project work assessment, and the "moving bar" of a first-time course.
2) Set up your personal GitHub portfolio from the course template, with a clean first commit and a .gitignore that keeps databases and secrets out of Git.
3) Set up the entire toolchain reproducibly with uv — one fixed Python plus SQLite, DuckDB, Datasette, and more — on Windows, macOS, or Linux, and verify it.
4) Explain — with a measurement, not a slide — the difference between an embedded database and a client/server database, and why most local apps should not run a server.
5) Load real rows into SQLite and query them from Python using bound parameters.
6) Take a first LLM-generated query, run it, and begin the habit that defines this course: verify before you trust.

## Learning Goals

> **How to use the learning goals.** Copy the module's learning goals from the course workbook into the block below. Keep the `LG1`, `LG2`, … IDs and a one-word keyword. In each task, write `Proves: LG2` (or link to `[LG2](#lg2)`) so you can find the evidence quickly at checkpoint time. In the **Evidence** line under each goal, link to the task that proves it.

<a id="lg1"></a>
### LG1 — `<CBL/PBL>`


Explain how this class works — the CBL/PBL philosophy, the portfolio-based project work assessment, and the "moving bar" of a first-time course

**Evidence:** [Task Inspirations](#task-inspirations)

<a id="lg2"></a>
### LG2 — `<GitHub portfolio>`

Set up your personal GitHub portfolio from the course template, with a clean first commit and a .gitignore that keeps databases and secrets out of Git.
## TEAM ##
Sara el hazmiri -https://github.com/selhazmiri

louay -https://github.com/louayXO

Favour Atem -https://github.com/Favouratem

John Manu Opoku



<a id="lg3"></a>
### LG3 — `<toolchain>`

## Set up the entire toolchain reproducibly with uv — one fixed Python plus SQLite, DuckDB, Datasette, and more — on Windows, macOS, or Linux, and verify it.## 

## Task 3: Install your whole toolchain with uv ##
Installing a toolchain with uv means using the uv package manager to create a reproducible Python development environment. You install uv, pin a specific Python version (3.14), create an isolated virtual environment, and install all required tools and libraries such as DuckDB, Datasette, sqlite-utils, Pydantic, and SQLAlchemy. uv automatically manages dependencies, generates lock files for reproducibility, and ensures the same setup works consistently across Windows, macOS, and Linux


 [## Task 3: Install your whole toolchain with uv ##](#task-what-is-iot)
![alt text](<Screenshot 2026-10-19 124449 - Copy.png>) UV installation in VS.code Using powershell 
![alt text](<Screenshot 2026-10-19 124449.png>)
![alt text](<Screenshot 2026-10-19 124518.png>)
![alt text](<Screenshot 2026-10-19 124740.png>)
![alt text](<Screenshot 2026-10-19 124612.png>)

[## Task 4: Meet the sqlite3 shell ##](#task-what-is-iot)
Description:
 Task 4 introduces you to the SQLite command-line shell and its special dot-commands (such as .tables, .schema, and .timer) that help you inspect and manage databases. The goal is to become familiar with the basic tools you will use throughout the course.

What you need to do:

Create and open a test database:

Shell
mkdir -p Module01
sqlite3 Module01/tour.db

Learning and using the essential shell commands:

.help → list available commands
.tables → show database tables
.schema reading → show table structure
.mode column and .headers on → improve output formatting
.timer on → measure query execution time
.quit → exit the shell


Creat a sample table and insert data using SQL:

SQL
CREATE TABLE reading(machine TEXT, vib REAL);
INSERT INTO reading VALUES ('cnc-01',1.2);

Run a query and view the results:

SQL
SELECT machine, MAX(vib)
FROM reading
GROUP BY machine;

Capture your session:

Create Module01/shell-tour.txt containing .tables, .schema, and a timed query.
Create Module01/tour.sql using the .dump command.
Write one sentence explaining which dot-command you found most useful and why.

![alt text](<Screenshot 2026-10-21 203232.png>) exploring the Dot command and SQL Commands 

![alt text](<Screenshot 2026-10-21 203232.png>)

DOt and SQL 
<style>
  a {
    text-decoration: none;
    color: #464feb;
  }
  tr th, tr td {
    border: 1px solid #e6e6e6;
    padding: 8px;
  }
  tr th {
    background-color: #f5f5f5;
  }
</style>

<table>
  <thead>
    <tr>
      <th>Dot Commands</th>
      <th>SQL Commands</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Start with a dot (<code>.</code>)</td>
      <td>End with a semicolon (<code>;</code>)</td>
    </tr>
    <tr>
      <td>Processed by the SQLite shell</td>
      <td>Processed by the SQLite database engine</td>
    </tr>
    <tr>
      <td>Used to manage or inspect the database environment</td>
      <td>Used to create, modify, and query data</td>
    </tr>
    <tr>
      <td>Work only inside the <code>sqlite3</code> command-line tool</td>
      <td>Work in almost any SQL database system</td>
    </tr>
    <tr>
      <td>Examples: <code>.tables</code>, <code>.schema</code>, <code>.help</code>, <code>.quit</code></td>
      <td>Examples: <code>SELECT</code>, <code>CREATE TABLE</code>, <code>INSERT</code>, <code>UPDATE</code></td>
    </tr>
  </tbody>
</table>

### Examples

**Dot command**

Shell

```text
.tables
```

Output:

```
reading
```

Shows the tables in the database.

**SQL command**

SQL

```sql
SELECT * FROM reading;
```

Output:

```
cnc-01 | 1.2
```

Retrieves data stored in the table.

### Easy way to remember

- **Dot commands = control the SQLite shell.**
- **SQL commands = work with the data inside the database.**

Think of it like this:

- `.tables` = "Show me what tables exist."
- `SELECT * FROM reading;` = "Show me the data inside the table."

[Task 5: Play SQL Island (~30 minutes)](#task-what-is-iot)
**SQL Island** is an interactive learning game for practicing SQL. You follow a story on an island and enter SQL queries to solve tasks using information stored in database tables. As you progress, you practice concepts such as `SELECT`, filtering with `WHERE`, sorting, and combining tables with joins.

Unlike SQLite dot-commands—which control the SQLite shell—SQL Island focuses on writing SQL statements to retrieve and work with data.

![alt text](<images/Screenshot 2026-10-28 224316.png>)
![alt text](<images/Screenshot 2026-10-28 224506.png>)
[Task 6: Load real telemetry and query it from Python](#task-what-is-iot)

![alt text](<../../images/Screenshot 2026-10-28 230511.png>)
![alt text](<Screenshot 2026-10-24 162627.png>)


<a id="lg4"></a>
### LG4 — `<embedded database and a client/server database>`

The cleanest way to explain the difference is with a **measurement of data access latency**.

Imagine a simple query:

> “Get customer #1234”.

### Embedded database

Examples: SQLite, DuckDB

The application and the database engine run in the **same process**.

```
App
 └─ SQLite library
     └─ local file
```

A request looks like:

```
Function call -> database engine -> file
```

Typical overhead:

- No network stack
- No TCP connection
- No serialisation/deserialisation
- No context switch to another server process

The extra cost is often measured in **microseconds (µs)**.

Example measurement:

```
Direct function call to embedded DB:
~10–100 µs overhead
```

---

### Client/server database

Examples: PostgreSQL, MySQL, SQL Server

The database runs as a separate service.

```
App  <----TCP/IP---->  Database Server
```

A request looks like:

```
Function call
→ serialise query
→ kernel/network stack
→ database server process
→ execute
→ serialise result
→ return
```

Even on the same machine ("localhost"), you still pay for IPC/network communication.

Typical overhead:

```
~0.2–2 ms
```

That's **hundreds to thousands of microseconds**.

| Architecture | Overhead per request |
|---|---:|
| Embedded DB | ~10–100 µs |
| Client/server DB | ~200–2000 µs |

The exact numbers vary, but the relationship is what matters:

**Client/server communication is usually one or more orders of magnitude slower than an in-process call.**

---

## Why most local apps should not run a server

For a desktop or local IoT application, a database server often solves problems you don't actually have.

### 1. More memory

Instead of:

```
myapp.exe
```

you now have:

```
myapp.exe
postgres.exe
```

or

```
myapp.exe
mysqld.exe
```

A separate server consumes RAM continuously, even when idle.

---

### 2. More CPU

The server must:

- manage connections
- schedule sessions
- handle authentication
- parse network messages
- maintain background workers

even though only one local application is using it.

---

### 3. More deployment complexity

With an embedded database you ship:

```
myapp.exe
data.db
```

With a server database you may need:

- service installation
- server configuration
- user accounts
- passwords
- upgrades
- backup configuration

More moving parts means more failures.

---

### 4. More security surface

A server typically exposes a port:

```
localhost:5432
localhost:3306
```

Now there is something listening for connections.

An embedded database usually exposes **nothing at all**.

---

### 5. No scalability benefit for a single-user app

The main reason to use a client/server database is:

- many users
- many machines
- remote access
- centralised administration

A notes app, personal dashboard, engineering tool, local IoT logger, or desktop game usually has none of these requirements.

---

## Rule of thumb

Use an **embedded database** when:

- one user
- one machine
- local data
- simple deployment

Use a **client/server database** when:

- multiple clients need simultaneous access
- data must be shared across machines
- central administration is required
- network access is a feature, not an accident

So the measurable difference is that an embedded database avoids the network/process boundary and typically reduces request overhead from **milliseconds to microseconds**. Most local applications gain faster access, lower resource usage, simpler deployment, and a smaller attack surface by using an embedded database instead of running a database server.



<a id="lg5"></a>
### LG5 — `<real rows>`

Load real rows into SQLite and query them from Python using bound parameters.

```python
import csv, math, random
from pathlib import Path
from datetime import datetime, timedelta

random.seed(42)
machines = ["cnc-01", "cnc-02", "cnc-03"]
start = datetime(2026, 1, 5, 6, 0, 0)          # Monday 06:00, shift start
days = 2
step = 3                                        # one reading every 3 s per machine
points = days * 24 * 60 * 60 // step            # 57,600 points x 3 machines

out = Path(__file__).with_name("telemetry.csv")
with out.open("w", newline="") as f:
    w = csv.writer(f)
    w.writerow(["ts", "machine", "temp_c", "vib_rms", "spindle_load"])
    for i in range(points):
        ts = (start + timedelta(seconds=i * step)).isoformat(sep=" ")
        for mach in machines:
            temp = 42 + 3 * math.sin(i / 500.0) + random.gauss(0, 0.6)
            vib = 1.2 + 0.4 * math.sin(i / 90.0) + random.gauss(0, 0.08)
            load = 60 + 15 * math.sin(i / 300.0) + random.gauss(0, 3)
            if mach == "cnc-02":                 # bearing wear: vibration creeps up over the 2 days
                vib += 0.9 * (i / points)
            if random.random() < 0.0008:         # rare tool-chatter spike
                vib += random.uniform(0.4, 0.9)
            w.writerow([ts, mach, f"{temp:.2f}", f"{vib:.3f}", f"{load:.1f}"])
print(f"wrote {out} ({out.stat().st_size/1e6:.1f} MB, {points * len(machines)} rows)")
```


**Evidence:** [Task What is IoT](#task-what-is-iot)
<a id="task-inspirations"></a>

<a id="lg6"></a>
### LG6 — `<LLM-generated query>`


Take a first LLM-generated query, run it, and begin the habit that defines this course: verify before you trust.

```sql
SELECT
    machine,
    AVG(spindle_load) AS avg_spindle_load
FROM readings
WHERE time(ts) >= '18:00:00'
GROUP BY machine
ORDER BY avg_spindle_load DESC;
```

### What to check in generated SQL

Generated SQL can look correct but still return misleading results. For this query:

- `time(ts) >= '18:00:00'` includes readings from 18:00 through midnight, but excludes early-morning readings. If “overnight” means 18:00–06:00, the filter must also include times before 06:00.
- Grouping only by `machine` combines matching readings across all dates in the table.
- `AVG` ignores `NULL` spindle-load values, so the average may use fewer rows than expected.
- Applying `time()` to `ts` can make it harder for SQLite to use a regular index on the timestamp column.

Always compare the query with the table schema and intended time range, then test the results against a few known rows. If a query is later built using user input, use bound parameters rather than inserting input directly into SQL.






## Reflection 1

Group Reflection on Module 01: Introduction and Local-First

As students of Industrial Engineering and Mechatronics, we found this module valuable because it connects database concepts to real engineering applications. Many of us are familiar with sensors, machines, production systems, and automation, but we had not fully considered how data should be stored, managed, and analysed efficiently.

One of the most important lessons was the concept of local-first systems. We learned that not every project requires a complex cloud infrastructure. For many engineering applications, such as machine monitoring, sensor data collection, laboratory measurements, or prototype development, embedded databases like SQLite can provide a simpler and more practical solution. This showed us the importance of selecting technologies based on actual requirements rather than following trends.

We were particularly interested in the examples related to manufacturing and industrial data. The idea that a single machine can generate useful operational data that can be stored and analysed locally helped us understand the relevance of databases in modern engineering environments. This aligns well with topics such as Industry 4.0, IoT systems, predictive maintenance, and process optimisation.

Another key takeaway was the emphasis on reproducibility and documentation. Using tools such as GitHub, SQLite, DuckDB, and uv encourages structured workflows that make projects easier to maintain, share, and reproduce. These practices are important not only in software development but also in engineering projects where reliability and traceability are essential.

Overall, this module provided a solid introduction to databases and demonstrated their role in engineering problem-solving. It helped us understand that effective data management is becoming an increasingly important skill for both Industrial Engineering and Mechatronics students as digital technologies continue to transform manufacturing and automation systems
