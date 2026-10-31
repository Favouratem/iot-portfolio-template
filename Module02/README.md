<style>
  a {
    text-decoration: none;
    color: #464feb;
  }
  tr th, tr td {
    border: 1px solid #e6e6e6;
  }
  tr th {
    background-color: #f5f5f5;
  }
</style>

# Module 2 — Data Contracts & Reading Legacy Schemas (ERD Audits)

## 👥 Team Context

This work was completed by a **student team consisting of Mechatronics Engineering and Industrial Engineering students**. We approached the maintenance database from both perspectives:

- **Mechatronics students** focused on machines, maintenance processes, spare parts, and technicians.
- **Industrial Engineering students** focused on data integrity, process reliability, information flow, and system standardisation.

---

## Table of Contents

- [Task 1: Reconstruct the ER Model](#task-1)
  - [Entity Analysis](#entity-analysis)
  - [Reconstructed ER Diagram](#reconstructed-er-diagram)
  - [Complete ER Diagram, SQL Schema, Queries, and Normalization](#machine-maintenance-database-complete-er-diagram-sql-schema-queries-and-normalization)
    - [Create the SQLite Database Schema](#2-create-the-sqlite-database-schema)
    - [Useful SQL Queries](#3-useful-sql-queries)
    - [Normalize the Database](#4-normalize-the-database)
    - [Check and Correct the ER Diagram](#5-check-and-correct-the-er-diagram)
  - [Cardinality Findings](#cardinality-findings)
- [Task 2: Integrity Defects](#task-2)
  - [Orphan Maintenance Jobs](#defect-1-orphan-maintenance-jobs)
  - [Duplicate Natural Keys](#defect-2-duplicate-natural-keys)
  - [Unit Price Stored as Text](#defect-3-unit-price-stored-as-text)
  - [Invalid Installed Dates](#defect-4-invalid-installed-dates)
  - [Non-Normalised Parts List](#defect-5-non-normalised-parts-list)
- [Task 3: Data Contract](#task-3)
  - [Design Decisions](#design-decisions)
  - [Corrected Schema Summary](#corrected-schema-summary)
  - [Expected DDL Output](#expected-ddl-output)
- [Task 4: Model Inheritance](#task-4)
  - [Scenario](#scenario)
  - [Supertype: Machine](#machine)
  - [Subtype: CNCMill](#subtype-cncmill)
  - [Subtype: InjectionMolder](#subtype-injectionmolder)
  - [EER Representation](#eer-representation)
- [Team Reflection](#team-reflection)

---

<a id="task-1"></a>
# ★ Task 1: Reconstruct the ER Model

## Entity Analysis

### 1. Machine

**Attributes**

- `machine_id` (int, surrogate key, intended)
- `machine_name`
- `serial_no`
- `site`
- `installed`

**Intended Primary Key**

- `machine_id` (surrogate key)
- `serial_no` should remain `UNIQUE`

---

### 2. Part

**Attributes**

- `part_no`
- `description`
- `unit_price`

**Intended Primary Key**

- `part_no` (natural key)

---

### 3. MaintenanceJob

**Attributes**

- `job_id`
- `machine_id`
- `technician_id`
- `done_on`

**Intended Primary Key**

- `job_id`

---

### 4. Technician

**Attributes**

- `technician_id`
- `technician_name`

**Intended Primary Key**

- `technician_id`

---

### 5. JobPart (Junction Table)

**Attributes**

- `job_id`
- `part_no`

**Primary Key**

- (`job_id`, `part_no`)

---

## Reconstructed ER Diagram

```text
Machine
  machine_id (PK)
  serial_no (UQ)
  site
  installed
      1
      |
      | performs maintenance on
      |
      N
MaintenanceJob
  job_id (PK)
  machine_id (FK)
  technician_id (FK)
  done_on
      1
      |
      | uses
      |
      N
JobPart
  job_id (FK)
  part_no (FK)
      N
      |
      | identifies
      |
      1
Part
  part_no (PK)
  description
  unit_price

Technician
  technician_id (PK)
  technician_name
      1
      |
      | performs
      |
      N

<a id="machine-maintenance-database-complete-er-diagram-sql-schema-queries-and-normalization"></a>
# Machine Maintenance Database: Complete ER Diagram, SQL Schema, Queries, and Normalization
We will work through all five tasks using your reconstructed database:

1. Draw the ER diagram.
2. Create the SQLite database schema.
3. Write useful SQL queries for maintenance records, technicians, spare parts, and costs.
4. Explain database normalization.
5. Check the ER diagram and correct any errors.

---

## 1. Draw the ER diagram
Your database contains five entities:

- **Machine** — stores information about each machine.
- **MaintenanceJob** — records maintenance performed on a machine.
- **Technician** — stores information about technicians.
- **Part** — stores information about spare parts.
- **JobPart** — links maintenance jobs to the parts used.

### 1.1 Corrected ER diagram
The following diagram shows the relationships and cardinalities.

```
+-----------------------+
|        MACHINE        |
+-----------------------+
| PK  machine_id        |
| UQ  serial_no         |
|     site              |
|     installed         |
+-----------------------+
           |
           | 1
           |
           | has
           |
           | N
+-----------------------+          +-----------------------+
|    MAINTENANCEJOB     |          |      TECHNICIAN       |
+-----------------------+          +-----------------------+
| PK  job_id            |          | PK  technician_id     |
| FK  machine_id        |          |     technician_name   |
| FK  technician_id     |          +-----------------------+
|     done_on           |                     |
+-----------------------+                     | 1
           |                                  |
           | 1                                | performs
           |                                  |
           | has                              | N
           |                                  |
           | N                                |
+-----------------------+                     |
|        JOBPART        |                     |
+-----------------------+                     |
| PK,FK job_id          |                     |
| PK,FK part_no         |                     |
+-----------------------+                     |
           |                                  |
           | N                                |
           |                                  |
           | uses                             |
           |                                  |
           | 1                                |
+-----------------------+                     |
|         PART          |                     |
+-----------------------+                     |
| PK  part_no           |                     |
|     description       |                     |
|     unit_price        |                     |
+-----------------------+
```

The diagram represents the following relationships:

RelationshipCardinalityExplanationMachine → MaintenanceJob1:NOne machine can have multiple maintenance jobs. Each job concerns one machine.Technician → MaintenanceJob1:NOne technician can perform multiple maintenance jobs. Each job is assigned to one technician.MaintenanceJob → JobPart1:NOne job can use multiple parts.Part → JobPart1:NOne part type can be used in multiple jobs.MaintenanceJob → PartM:NA maintenance job can use multiple parts, and a part can be used in multiple jobs. `JobPart` resolves this many-to-many relationship.

**Important:** `JobPart` is an associative entity. Its composite primary key is `(job_id, part_no)` if each part type can appear only once per job. If a job can contain multiple separate entries for the same part, the design should include a quantity or a separate line identifier.

---

## 2. Create the SQLite database schema
Now we will convert the ER diagram into actual SQLite tables.

### 2.1 SQL table definitions
The schema below includes primary keys, foreign keys, unique constraints, and appropriate data types. I have also added a `quantity` column to `JobPart` so that we can calculate the cost of parts used in each maintenance job.

```
PRAGMA foreign_keys = ON;

-- 1. Machine table
CREATE TABLE Machine (
    machine_id INTEGER PRIMARY KEY,
    serial_no TEXT NOT NULL UNIQUE,
    site TEXT NOT NULL,
    installed TEXT
);

-- 2. Technician table
CREATE TABLE Technician (
    technician_id INTEGER PRIMARY KEY,
    technician_name TEXT NOT NULL
);

-- 3. Part table
CREATE TABLE Part (
    part_no TEXT PRIMARY KEY,
    description TEXT NOT NULL,
    unit_price REAL NOT NULL CHECK (unit_price >= 0)
);

-- 4. MaintenanceJob table
CREATE TABLE MaintenanceJob (
    job_id INTEGER PRIMARY KEY,
    machine_id INTEGER NOT NULL,
    technician_id INTEGER NOT NULL,
    done_on TEXT NOT NULL,

    FOREIGN KEY (machine_id)
        REFERENCES Machine(machine_id),

    FOREIGN KEY (technician_id)
        REFERENCES Technician(technician_id)
);

-- 5. JobPart table
CREATE TABLE JobPart (
    job_id INTEGER NOT NULL,
    part_no TEXT NOT NULL,
    quantity INTEGER NOT NULL DEFAULT 1
        CHECK (quantity > 0),

    PRIMARY KEY (job_id, part_no),

    FOREIGN KEY (job_id)
        REFERENCES MaintenanceJob(job_id),

    FOREIGN KEY (part_no)
        REFERENCES Part(part_no)
);
```

### 2.2 Explanation of the schema
**Primary keys (PK)** uniquely identify each record.

For example:

```
machine_id INTEGER PRIMARY KEY
```

Each machine receives a unique identifier.

**Foreign keys (FK)** connect records across tables.

For example:

```
FOREIGN KEY (machine_id)
    REFERENCES Machine(machine_id)
```

This ensures that a maintenance job cannot reference a machine that does not exist, provided foreign-key enforcement is enabled on the database connection.

**Unique constraints (UQ)** prevent duplicate values.

```
serial_no TEXT NOT NULL UNIQUE
```

Two machines cannot have the same serial number.

**The composite primary key in `JobPart`:**

```
PRIMARY KEY (job_id, part_no)
```

This prevents the same part number from appearing more than once in a job. The `quantity` column records how many units of that part were used.

For this simple schema, `quantity` represents the total quantity of each part type used in a job.

**SQLite consideration:** `PRAGMA foreign_keys = ON;` should be executed for every database connection that needs foreign-key enforcement. The `installed` and `done_on` fields use `TEXT`, allowing dates to be stored in ISO format, such as `2026-10-29` or `2026-10-29 14:30:00`.

### 2.3 Example data
You can populate the tables with the following sample records to test the schema.

```
-- Machines
INSERT INTO Machine
    (machine_id, serial_no, site, installed)
VALUES
    (1, 'MCH-1001', 'Factory A', '2024-01-15'),
    (2, 'MCH-1002', 'Factory B', '2023-06-20');

-- Technicians
INSERT INTO Technician
    (technician_id, technician_name)
VALUES
    (1, 'John Smith'),
    (2, 'Sarah Johnson');

-- Parts
INSERT INTO Part
    (part_no, description, unit_price)
VALUES
    ('P001', 'Bearing', 45.00),
    ('P002', 'Drive belt', 25.00),
    ('P003', 'Lubrication filter', 15.00);

-- Maintenance jobs
INSERT INTO MaintenanceJob
    (job_id, machine_id, technician_id, done_on)
VALUES
    (101, 1, 1, '2026-10-20'),
    (102, 1, 2, '2026-10-25'),
    (103, 2, 1, '2026-10-26');

-- Parts used in each job
INSERT INTO JobPart
    (job_id, part_no, quantity)
VALUES
    (101, 'P001', 2),
    (101, 'P003', 1),
    (102, 'P002', 1),
    (103, 'P001', 1);
```

The database now contains two machines, two technicians, three part types, three maintenance jobs, and four job-part records.

---

## 3. Useful SQL queries
The following queries demonstrate how to extract useful information from the database.

### Query 1: Display all maintenance jobs
Retrieve each maintenance job together with its machine and assigned technician.

```
SELECT
    mj.job_id,
    m.serial_no,
    m.site,
    t.technician_name,
    mj.done_on
FROM MaintenanceJob AS mj
JOIN Machine AS m
    ON mj.machine_id = m.machine_id
JOIN Technician AS t
    ON mj.technician_id = t.technician_id
ORDER BY mj.done_on DESC;
```

**What this does:**

- Joins `MaintenanceJob` to `Machine` using `machine_id`.
- Joins `MaintenanceJob` to `Technician` using `technician_id`.
- Displays the most recent maintenance jobs first.

### Query 2: Find the maintenance history of a specific machine
Suppose you want to inspect machine `MCH-1001`.

```
SELECT
    m.serial_no,
    mj.job_id,
    mj.done_on,
    t.technician_name
FROM Machine AS m
JOIN MaintenanceJob AS mj
    ON m.machine_id = mj.machine_id
JOIN Technician AS t
    ON mj.technician_id = t.technician_id
WHERE m.serial_no = 'MCH-1001'
ORDER BY mj.done_on DESC;
```

This retrieves all recorded maintenance jobs for that machine.

### Query 3: Count maintenance jobs performed by each technician

```
SELECT
    t.technician_id,
    t.technician_name,
    COUNT(mj.job_id) AS total_jobs
FROM Technician AS t
LEFT JOIN MaintenanceJob AS mj
    ON t.technician_id = mj.technician_id
GROUP BY
    t.technician_id,
    t.technician_name
ORDER BY total_jobs DESC;
```

**Why use `LEFT JOIN`?**

It includes technicians who have not yet performed any maintenance jobs. Their job count will be zero.

### Query 4: Calculate the total parts cost for each maintenance job

```
SELECT
    mj.job_id,
    mj.done_on,
    COALESCE(
        SUM(jp.quantity * p.unit_price),
        0
    ) AS total_parts_cost
FROM MaintenanceJob AS mj
LEFT JOIN JobPart AS jp
    ON mj.job_id = jp.job_id
LEFT JOIN Part AS p
    ON jp.part_no = p.part_no
GROUP BY
    mj.job_id,
    mj.done_on
ORDER BY total_parts_cost DESC;
```

The calculation is:

\[
\text{Parts Cost} =
\sum (\text{Quantity Used} \times \text{Unit Price})
\]For example, job `101` used two bearings costing €45 each and one filter costing €15.

\[
\text{Cost}=(2\times45)+(1\times15)
\]\[
\boxed{\text{Total parts cost}=€105}
\]This query calculates the cost of parts only. It does not include technician labour, downtime, or other maintenance expenses.

**Important:** This query uses the current price recorded in `Part`. If part prices change over time, you should store the actual unit price charged at the time of each maintenance job to preserve historical cost accuracy.

### Query 5: Identify the machine with the most maintenance jobs

```
SELECT
    m.machine_id,
    m.serial_no,
    m.site,
    COUNT(mj.job_id) AS total_maintenance_jobs
FROM Machine AS m
LEFT JOIN MaintenanceJob AS mj
    ON m.machine_id = mj.machine_id
GROUP BY
    m.machine_id,
    m.serial_no,
    m.site
ORDER BY total_maintenance_jobs DESC
LIMIT 1;
```

This identifies the machine with the highest number of recorded maintenance jobs.

If several machines have the same highest count, this query returns just one of them. To return every machine tied for the highest count, a different query would be needed.

### Query 6: List the parts used for each maintenance job

```
SELECT
    mj.job_id,
    mj.done_on,
    p.part_no,
    p.description,
    jp.quantity,
    p.unit_price,
    jp.quantity * p.unit_price AS line_cost
FROM MaintenanceJob AS mj
JOIN JobPart AS jp
    ON mj.job_id = jp.job_id
JOIN Part AS p
    ON jp.part_no = p.part_no
ORDER BY
    mj.job_id,
    p.part_no;
```

This query provides a detailed breakdown of parts used, including the quantity, unit price, and line cost.

### Query 7: Calculate the total parts expenditure across all maintenance jobs

```
SELECT
    COALESCE(
        SUM(jp.quantity * p.unit_price),
        0
    ) AS total_parts_expenditure
FROM JobPart AS jp
JOIN Part AS p
    ON jp.part_no = p.part_no;
```

This calculates the combined cost of all parts recorded in `JobPart`.

With the example data, the total is:

\[
105+25+45=\boxed{€175}
\]This assumes the sample unit prices are in euros.

---

## 4. Normalize the database
Database normalization is the process of organizing data to reduce unnecessary duplication and prevent data anomalies.

For your maintenance database, normalization helps ensure that machine details, technician details, part prices, and maintenance records are stored in the appropriate tables.

We will examine the first three normal forms: **1NF, 2NF, and 3NF**.

### 4.1 First Normal Form (1NF)
A table is in First Normal Form when:

- Each column contains a single value in each row.
- Each record can be uniquely identified.
- Repeating groups are not stored in a single field.
**Example of an unnormalized table**

Imagine storing all the parts used in a maintenance job in one field:

job_idmachine_idtechnicianparts_used1011John SmithBearing, Filter1021Sarah JohnsonDrive belt1032John SmithBearing

The `parts_used` column contains multiple values in a single cell.

This makes it difficult to search for individual parts, calculate their costs, or record quantities reliably.

**Solution:** Create a separate `JobPart` table.

job_idpart_noquantity101P0012101P0031102P0021103P0011

Each row now records one part type used in one maintenance job.

This structure satisfies the relevant 1NF requirements.

### 4.2 Second Normal Form (2NF)
A table is in Second Normal Form if:

1. It is already in 1NF.
2. Every non-key attribute depends on the whole candidate key, not just part of a composite key.
Consider the `JobPart` table.

Its primary key is:

\[
(\text{job\_id},\text{part\_no})
\]Suppose we added the part description and price directly to this table:

job_idpart_nodescriptionunit_pricequantity101P001Bearing45.002101P003Filter15.001102P002Drive belt25.001

The problem is that:

- `description` depends only on `part_no`.
- `unit_price` depends only on `part_no`.
- `quantity` depends on the combination of `job_id` and `part_no`.
The description and unit price therefore depend on only part of the composite key.

This creates a partial dependency.

**Solution:** Store part descriptions and prices in the `Part` table and keep `JobPart` for the relationship between jobs and parts.

The resulting structure is:

**Part**

part_no (PK)descriptionunit_priceP001Bearing45.00P002Drive belt25.00P003Lubrication filter15.00

**JobPart**

job_id (PK, FK)part_no (PK, FK)quantity101P0012101P0031102P0021

Now, the part details are stored once, and the quantity is associated with the complete composite key.

Both tables satisfy the relevant 2NF requirements.

### 4.3 Third Normal Form (3NF)
A table is in Third Normal Form if:

1. It is already in 2NF.
2. Non-key attributes do not depend transitively on a key through another non-key attribute.
Consider this hypothetical table:

job_idmachine_idserial_nosite1011MCH-1001Factory A1021MCH-1001Factory A1032MCH-1002Factory B

Assume the following dependencies:

\[
\text{job\_id}\rightarrow\text{machine\_id}
\]and

\[
\text{machine\_id}\rightarrow
\{\text{serial\_no},\text{site}\}
\]Consequently, the machine's serial number and site are transitively dependent on the maintenance job's primary key.

Storing those details in every maintenance record would duplicate information.

**Solution:** Store machine attributes in `Machine` and reference the machine through `machine_id` in `MaintenanceJob`.

This removes the transitive dependency from the maintenance-job table.

### 4.4 Summary of normalization
Normal formMain requirementHow your design satisfies it1NFAtomic values and no repeating groupsIndividual part records are stored in `JobPart`.2NFNo partial dependency on a composite candidate keyPart descriptions and prices are stored in `Part`.3NFNo transitive dependencies among non-key attributesMachine and technician details are stored in their respective tables.

**Conclusion:** Your five-table design is consistent with 3NF, assuming the stated dependencies and keys are correct.

Normalization also reduces the risk of:

- **Update anomalies:** Changing a machine's site in one place but not another.
- **Insertion anomalies:** Being unable to record a new part until it has been used in a job.
- **Deletion anomalies:** Accidentally losing part information when deleting a maintenance record.

---

## 5. Check and correct the ER diagram
Your reconstructed ER diagram is fundamentally sound. However, a few details should be clarified to ensure that the database represents the real-world maintenance process correctly.

### 5.1 Verify each relationship

#### Relationship A: Machine to MaintenanceJob
**Your design:**

```
Machine (1) ───────── (N) MaintenanceJob
```

This is correct if each maintenance job concerns exactly one machine.

A machine may have no maintenance history yet, so the relationship is more precisely expressed as:

- One machine can have zero or many maintenance jobs.
- Each maintenance job must refer to exactly one machine.
The foreign key is:

```
MaintenanceJob.machine_id
    REFERENCES Machine(machine_id)
```

#### Relationship B: Technician to MaintenanceJob
**Your design:**

```
Technician (1) ───────── (N) MaintenanceJob
```

This is correct if every maintenance job is assigned to exactly one technician.

However, consider the real-world situation: can two or more technicians work on the same maintenance job?

If the answer is yes, your current model would be too restrictive because `MaintenanceJob` contains only one `technician_id`.

In that case, introduce an associative entity:

```
Technician
    1
    |
    N
MaintenanceJobTechnician
    N
    |
    1
MaintenanceJob
```

The new table would contain:

```
CREATE TABLE MaintenanceJobTechnician (
    job_id INTEGER NOT NULL,
    technician_id INTEGER NOT NULL,

    PRIMARY KEY (job_id, technician_id),

    FOREIGN KEY (job_id)
        REFERENCES MaintenanceJob(job_id),

    FOREIGN KEY (technician_id)
        REFERENCES Technician(technician_id)
);
```

This supports multiple technicians per job.

**Recommendation:** Keep your original relationship if the business rule guarantees one technician per maintenance job.

#### Relationship C: MaintenanceJob to Part
Your original diagram represents this as a many-to-many relationship resolved through `JobPart`.

```
MaintenanceJob (1) ─── (N) JobPart (N) ─── (1) Part
```

This is correct.

For example, maintenance job `101` may use two bearings and one lubrication filter.

The `JobPart` table records both part types and their quantities.

There is one important business rule to consider: the composite primary key `(job_id, part_no)` permits only one row for a particular part number within each job.

That is appropriate if you aggregate the total quantity of each part into one row. If you need multiple separate usage entries for the same part, you should introduce a separate `job_part_id` or usage-event identifier.

### 5.2 Verify primary keys and foreign keys
Your keys should be defined as follows:

TablePrimary keyForeign keysOther constraintsMachine`machine_id`None`serial_no` must be uniqueTechnician`technician_id`None`technician_name` should be requiredMaintenanceJob`job_id``machine_id`, `technician_id`Machine and technician references should be validPart`part_no`None`unit_price` should be non-negativeJobPart`(job_id, part_no)``job_id`, `part_no``quantity` should be positive

These definitions are consistent with the SQL schema in Section 2.

### 5.3 Consider storing the price paid at the time of maintenance
There is one additional improvement worth considering.

Suppose a bearing costs €45 when maintenance job `101` takes place. Six months later, its price increases to €55.

If you store only the current price in `Part.unit_price`, recalculating the historical cost of job `101` would incorrectly use €55.

For accurate historical reporting, you could add a price snapshot to `JobPart`:

```
CREATE TABLE JobPart (
    job_id INTEGER NOT NULL,
    part_no TEXT NOT NULL,
    quantity INTEGER NOT NULL DEFAULT 1
        CHECK (quantity > 0),
    unit_price_at_job REAL NOT NULL
        CHECK (unit_price_at_job >= 0),

    PRIMARY KEY (job_id, part_no),

    FOREIGN KEY (job_id)
        REFERENCES MaintenanceJob(job_id),

    FOREIGN KEY (part_no)
        REFERENCES Part(part_no)
);
```

With this design, `Part.unit_price` represents the current standard price, while `JobPart.unit_price_at_job` preserves the price charged for that particular maintenance job.

You would then calculate historical expenditure using:

```
SELECT
    job_id,
    SUM(quantity * unit_price_at_job) AS total_parts_cost
FROM JobPart
GROUP BY job_id;
```

**Note:** This is an alternative definition of `JobPart`; use it instead of the earlier table definition, not alongside it. When inserting job-part records, your application must capture the applicable price at the time of the job.

### 5.4 Final assessment
Your reconstructed ER model is correct for a maintenance system with these business rules:

1. Every maintenance job concerns one machine.
2. Every maintenance job is assigned to one technician.
3. A machine can have multiple maintenance jobs.
4. A technician can perform multiple maintenance jobs.
5. A maintenance job can use multiple parts.
6. A part can be used in multiple maintenance jobs.
The five-table structure is a suitable relational design, and the separation between `MaintenanceJob`, `Part`, and `JobPart` correctly resolves the many-to-many relationship.

The main decisions still to confirm are whether multiple technicians can work on one job, whether repeated part-usage entries must be recorded separately, and whether historical part prices must be preserved.

### Recommended next step
Implement the schema in SQLite, insert the sample records, and run the queries in Section 3. This will let you validate the relationships and check that the maintenance reports return the expected results before using the database with real machine data.
MaintenanceJob
```

## Cardinality Findings

### Query

```sql
SELECT
    job_id,
    parts_used,
    LENGTH(parts_used) - LENGTH(REPLACE(parts_used, ',', '')) + 1 AS n_parts
FROM maint_job;
```

### Result

| job_id | parts_used | n_parts |
|---:|---|---:|
| 1 | P-1001,P-1002 | 2 |
| 2 | P-1003 | 1 |
| 3 | P-1001 | 1 |
| 4 | NULL | NULL |

## Cardinality Conclusions

### 1:1 Relationships

No true 1:1 relationship is explicitly modelled in the legacy schema.

---

### 1:n Relationships

One machine can have many maintenance jobs.

Example:

```text
cnc-01 → Job 1
cnc-01 → Job 4
```

The schema partially models this through `machine_name`.

---

### m:n Relationships

Many maintenance jobs can use many parts.

Example:

```text
Job 1 uses P-1001 and P-1002
P-1001 can appear in multiple jobs
```

The schema hides this relationship inside the `parts_used` text column, violating First Normal Form (1NF).

---

<a id="task-2"></a>
# ★ Task 2: Integrity Defects

---

## Defect 1: Orphan Maintenance Jobs

### Query

```sql
SELECT m.job_id, m.machine_name
FROM maint_job m
LEFT JOIN machine x
    ON m.machine_name = x.machine_name
WHERE x.machine_name IS NULL;
```

### Result

| job_id | machine_name |
|---:|---|
| 3 | cnc-05 |

### Production Impact

Maintenance history cannot be linked to a real machine, leading to inaccurate maintenance planning and reporting.

---

## Defect 2: Duplicate Natural Keys

### Query

```sql
SELECT part_no, COUNT(*) AS copies
FROM part
GROUP BY part_no
HAVING COUNT(*) > 1;
```

### Result

| part_no | copies |
|---|---:|
| P-1003 | 2 |

### Production Impact

Duplicate spare-part records can cause incorrect inventory counts and purchasing decisions.

---

## Defect 3: Unit Price Stored as Text

### Query

```sql
SELECT part_no, unit_price
FROM part
WHERE unit_price < '15';
```

### Result

Results depend on text comparison rather than numeric comparison.

### Production Impact

Invoice calculations and cost analyses may produce incorrect results.

---

## Defect 4: Invalid Installed Dates

### Query

```sql
SELECT machine_name, installed
FROM machine
WHERE installed NOT GLOB
    '[0-9][0-9][0-9][0-9]-[0-9][0-9]-[0-9][0-9]';
```

### Result

| machine_name | installed |
|---|---|
| RANDOM-NAME | unknown |

### Production Impact

Machine age and lifecycle calculations become unreliable.

---

## Defect 5: Non-Normalised Parts List

### Query

```sql
SELECT j.job_id, p.description
FROM maint_job j
JOIN part p
    ON p.part_no = j.parts_used;
```

### Result

No rows returned.

### Production Impact

The company cannot reliably determine which parts were used in maintenance activities.

---

<a id="task-3"></a>
# ★ Task 3: Data Contract

## Design Decisions

### Constraints Enforced in Pydantic

The application layer should validate:

- serial number format (`SN-xxxx`)
- valid dates
- list structure of part numbers
- minimum field lengths

**Reason:** Invalid user input should be rejected before reaching the database.

---

### Constraints Enforced in SQLAlchemy / Database

The database should enforce:

- Primary Keys
- Foreign Keys
- `UNIQUE` constraints
- `NOT NULL` constraints
- Numeric data types

**Reason:** Database rules guarantee data integrity regardless of the application used.

---

## Corrected Schema Summary

### Machine

```text
machine_id (PK)
serial_no UNIQUE
site NOT NULL
installed DATE NOT NULL
```

### Part

```text
part_no (PK)
description NOT NULL
unit_price NUMERIC(9,2)
```

### MaintenanceJob

```text
job_id (PK)
machine_id (FK)
technician
done_on DATE
```

### JobPart

```text
job_id (FK)
part_no (FK)
PRIMARY KEY(job_id, part_no)
```

---

## Expected DDL Output

```sql
CREATE TABLE machine (
    machine_id INTEGER PRIMARY KEY,
    serial_no VARCHAR NOT NULL,
    site VARCHAR NOT NULL,
    installed DATE NOT NULL,
    CONSTRAINT uq_machine_serial UNIQUE (serial_no)
);

CREATE TABLE part (
    part_no VARCHAR PRIMARY KEY,
    description VARCHAR NOT NULL,
    unit_price NUMERIC(9,2) NOT NULL
);

CREATE TABLE maint_job (
    job_id INTEGER PRIMARY KEY,
    machine_id INTEGER NOT NULL,
    technician VARCHAR NOT NULL,
    done_on DATE NOT NULL,
    FOREIGN KEY(machine_id)
        REFERENCES machine(machine_id)
);

CREATE TABLE job_part (
    job_id INTEGER,
    part_no VARCHAR,
    PRIMARY KEY(job_id, part_no),
    FOREIGN KEY(job_id)
        REFERENCES maint_job(job_id),
    FOREIGN KEY(part_no)
        REFERENCES part(part_no)
);
```

---

<a id="task-4"></a>
# ◇ Task 4: Model Inheritance

## Scenario

The company operates multiple machine types:

- CNC Mills
- Injection Molders

Both are machines but have specialised attributes.

---

## Supertype

### Machine

```text
machine_id
serial_no
site
installed
```

---

## Subtype: CNCMill

```text
machine_id
number_of_axes
max_rpm
```

---

## Subtype: InjectionMolder

```text
machine_id
clamping_force
shot_size
```

---

## EER Representation

```text
Machine
   |
   +---- CNCMill
   |
   +---- InjectionMolder
```

This is an example of **generalisation / inheritance**, where CNC mills and injection molders inherit common machine attributes.

---

<a id="team-reflection"></a>
# Reflection

## Team Reflection

As a mixed team of Mechatronics and Industrial Engineering students, this exercise showed us how strongly physical maintenance systems depend on high-quality data models.

From the Mechatronics perspective, we recognised that machines, spare parts, and maintenance activities have clear real-world relationships that should be reflected accurately in a database. Poor schema design can make it difficult to trace failures, analyse machine performance, or plan maintenance activities.

From the Industrial Engineering perspective, we learned that data quality directly affects operational efficiency, inventory management, reporting accuracy, and decision-making. Data integrity issues such as duplicate records, orphan references, and invalid data types can create significant business risks.

A key insight was that an ER diagram is not enough on its own. Real systems require enforceable contracts through validation rules, foreign keys, unique constraints, and proper normalisation. We also learned how legacy systems often hide business rules inside text fields, requiring systematic auditing to reconstruct the original design intent.

Overall, this exercise improved our understanding of conceptual database modelling, data governance, system reliability, and the practical challenges of working with legacy industrial databases.
