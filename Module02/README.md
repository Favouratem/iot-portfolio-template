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

# Reflection

## Team Reflection

As a mixed team of Mechatronics and Industrial Engineering students, this exercise showed us how strongly physical maintenance systems depend on high-quality data models.

From the Mechatronics perspective, we recognised that machines, spare parts, and maintenance activities have clear real-world relationships that should be reflected accurately in a database. Poor schema design can make it difficult to trace failures, analyse machine performance, or plan maintenance activities.

From the Industrial Engineering perspective, we learned that data quality directly affects operational efficiency, inventory management, reporting accuracy, and decision-making. Data integrity issues such as duplicate records, orphan references, and invalid data types can create significant business risks.

A key insight was that an ER diagram is not enough on its own. Real systems require enforceable contracts through validation rules, foreign keys, unique constraints, and proper normalisation. We also learned how legacy systems often hide business rules inside text fields, requiring systematic auditing to reconstruct the original design intent.

Overall, this exercise improved our understanding of conceptual database modelling, data governance, system reliability, and the practical challenges of working with legacy industrial databases.
