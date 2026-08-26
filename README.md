# E-R Modelling

Notes and examples from learning database design — from system analysis to E-R diagrams to Prisma schema. Tech stack: PostgreSQL + Prisma.

---

## Why System Analysis First

Before designing tables, requirements need to be read carefully to find entities, attributes, and how they relate — this is **system analysis**. It's followed by **logical modelling** (E-R diagrams, tech-agnostic) and then **physical modelling** (actual database schema). Skipping straight to tables is how you end up missing junction tables or breaking relationships.

## Crow's Foot Notation

| Symbol | Meaning |
|---|---|
| `\|\|` | Exactly one |
| `o\|` | Zero or one |
| `\|{` | One or many |
| `o{` | Zero or many |

## Relationship Types

- **1:1** — e.g. Patient ↔ EmergencyContact
- **1:N** — e.g. Publisher → Book
- **M:N** — e.g. Student ↔ Subject (needs a junction table, since a column can only hold one FK value)

## Reading a Table (Entity Box)

Each box in a diagram = one table. Inside it:

- **PK (Primary Key)** — the column that uniquely identifies each row, like a fingerprint. No two rows share the same PK.
- **FK (Foreign Key)** — a column that stores another table's PK, used to link two tables together.
- Every other field is just a regular attribute (name, date, phone, etc).

For example, in `TREATMENT` below, `doctor_id` and `patient_id` are both FKs — they're what tie one Doctor and one Patient together for that specific record.

### Example: Hospital System

**Problem statement:** *A hospital has doctors. Each doctor can treat many patients, and each patient can be treated by many doctors over time. Each patient has exactly one primary emergency contact person.*

From this: Doctor↔Patient is M:N (both sides say "many"), and Patient↔EmergencyContact is 1:1 (both sides say "one").

```mermaid
erDiagram
  DOCTOR ||--o{ TREATMENT : gives
  PATIENT ||--o{ TREATMENT : receives
  PATIENT ||--|| EMERGENCYCONTACT : has
  DOCTOR {
    int id PK
    string name
    string specialty
  }
  PATIENT {
    int id PK
    string name
    string dob
  }
  TREATMENT {
    int id PK
    int doctor_id FK
    int patient_id FK
    string date
  }
  EMERGENCYCONTACT {
    int id PK
    int patient_id FK
    string name
    string phone
  }
```

Doctor ↔ Patient is M:N, resolved with the `Treatment` junction table. Patient ↔ EmergencyContact is 1:1, so the FK just sits on `EmergencyContact` with a unique constraint — no junction table needed.

## Prisma Translation

Each entity box becomes a `model`. Each FK becomes a scalar field + a `@relation`. Here's how the hospital diagram maps over, piece by piece:

**Doctor & Patient** — plain entities to start, plus a `Treatment[]` field. That array isn't a real column; it just lets Prisma fetch `doctor.treatments` in code.

```prisma
model Doctor {
  id         Int         @id @default(autoincrement())
  name       String
  specialty  String
  treatments Treatment[]
}

model Patient {
  id               Int               @id @default(autoincrement())
  name             String
  dob              DateTime
  treatments       Treatment[]
  emergencyContact EmergencyContact?
}
```

**Treatment** — the junction table for the M:N. It holds two FKs, one to each side. This is what turns "many doctors ↔ many patients" into two ordinary 1:N relationships.

```prisma
model Treatment {
  id        Int      @id @default(autoincrement())
  date      DateTime
  doctorId  Int
  doctor    Doctor   @relation(fields: [doctorId], references: [id])
  patientId Int
  patient   Patient  @relation(fields: [patientId], references: [id])
}
```

**EmergencyContact** — the 1:1 side. Same FK pattern as above, but `patientId` is marked `@unique`, which is the one thing stopping a patient from having more than one contact.

```prisma
model EmergencyContact {
  id        Int     @id @default(autoincrement())
  name      String
  phone     String
  patientId Int     @unique
  patient   Patient @relation(fields: [patientId], references: [id])
}
```

**Key takeaway:** M:N → separate junction model with two FKs. 1:1 → FK on the dependent model with `@unique`. 1:N → FK on the "many" model, no `@unique`.

---
# Normalization

Normalization is a set of rules for structuring tables so that fixing one fact means updating exactly one row — avoiding **update, delete, and insert anomalies** caused by duplicated data.

**Example problem** — a single unnormalized table:

| student_id | subject | teacher_name | teacher_phone |
|---|---|---|---|
| 1 | Math | Mr. Sharma | 98765 |
| 1 | Nepali | Mr. Sharma | 98765 |
| 2 | Math | Mr. Sharma | 98765 |

If Mr. Sharma changes his phone number, it has to be updated in every row he appears in — miss one, and the data contradicts itself.

### 1NF — Atomic Columns

Every column holds one single value; no repeating groups like `subject1, subject2, subject3`. A comma-separated list in one column (e.g. `"Math, Nepali"`) breaks 1NF — it should be split into one row per value.

### 2NF — Full Dependency on the Whole Key

Applies only when the primary key is composite (e.g. `(student_id, subject)`). Every other column must depend on the **entire** key, not just part of it. Above, `teacher_phone` depends only on `subject`, not on `student_id` — a **partial dependency**. Fix: move `subject` + `teacher_phone` into their own table.

### 3NF — No Transitive Dependencies

Every column must depend directly on the key — not on another non-key column. Above, `teacher_phone` depends on `teacher_name`, which is itself just a regular column, not the key. Fix: pull `Teacher` into its own table entirely.

```prisma
model Subject {
  id        Int     @id @default(autoincrement())
  name      String
  teacherId Int
  teacher   Teacher @relation(fields: [teacherId], references: [id])
}

model Teacher {
  id       Int       @id @default(autoincrement())
  name     String
  phone    String
  subjects Subject[]
}
```

Now `Mr. Sharma`'s phone number exists in exactly one row, regardless of how many subjects he teaches.

**One-line summary:** *Every non-key column must depend on the key, the whole key, and nothing but the key.*

### BCNF — A Stricter 3NF

For every dependency `X → Y`, `X` must be a candidate key on its own. This only diverges from 3NF in edge cases with overlapping composite keys — rare in typical CRUD apps, but good to recognize by name.

### Denormalization — Breaking the Rules on Purpose

Full normalization means more joins to read combined data, which costs performance. Denormalization deliberately duplicates data to avoid that cost — valid when done intentionally, not by accident. Classic example: storing `priceAtPurchase` on an order line even though `Product.price` already exists elsewhere, because the order should freeze the price at that point in time, not reflect future price changes.

| Form | Rule |
|---|---|
| 1NF | Atomic columns, no repeating groups |
| 2NF | No partial dependency (composite keys only) |
| 3NF | No transitive dependency |
| BCNF | Every determinant is a candidate key |
| Denormalization | Intentional duplication for read performance/history |

---

*Next: keys & constraints, indexing, ACID, joins.*