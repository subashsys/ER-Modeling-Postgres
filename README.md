# E-R Modelling

Notes and examples from learning database design — from system analysis to E-R diagrams to Prisma schema. Tech stack: PostgreSQL + Prisma.

---
![Movie Ticket Booking ERD](./images/erd.png)

## Movie Ticket Booking System

**Problem statement:** *A cinema has multiple screens (halls). Each screen shows movies at scheduled showtimes. A customer can book a ticket for a specific showtime, choosing one or more seats. Each screen has a fixed set of seats. A seat can only be booked once per showtime — it can't be double-booked. A movie can be shown on many different screens at many different times, and a screen hosts many different movies across the week. Each booking should also record the payment amount and status (paid/pending/failed).*

**Reasoning:**
- `Movie ↔ Screen` is M:N, but no new junction table is needed — `Showtime` already sits between them (it holds `movieId` + `screenId`, plus its own `date`/`startTime`), so it naturally does the job of a junction table.
- `Booking ↔ Seat` is also M:N, and needs a dedicated junction table (`SeatBooking`). Since the real business rule is "no seat double-booked for the same *showtime*," `showtimeId` is duplicated onto `SeatBooking` itself — a `@@unique` constraint can only be enforced on columns physically present on that table, not across a join.
- `Booking ↔ Payment` is 1:N, not 1:1 — a booking can have a failed payment attempt followed by a successful retry, and both need to be recorded.

```prisma
model Customer {
  id       Int       @id @default(autoincrement())
  name     String
  email    String    @unique
  phone    String
  bookings Booking[]
}

model Movie {
  id              Int        @id @default(autoincrement())
  title           String
  durationMinutes Int
  language        String
  showtimes       Showtime[]
}

model Screen {
  id         Int        @id @default(autoincrement())
  name       String
  totalSeats Int
  seats      Seat[]
  showtimes  Showtime[]
}

model Seat {
  id           Int           @id @default(autoincrement())
  seatNumber   String
  screenId     Int
  screen       Screen        @relation(fields: [screenId], references: [id])
  seatBookings SeatBooking[]

  @@unique([screenId, seatNumber])   // no duplicate seat labels on the same screen
}

model Showtime {
  id           Int           @id @default(autoincrement())
  date         DateTime
  startTime    DateTime
  movieId      Int
  movie        Movie         @relation(fields: [movieId], references: [id])
  screenId     Int
  screen       Screen        @relation(fields: [screenId], references: [id])
  bookings     Booking[]
  seatBookings SeatBooking[]
}

model Booking {
  id           Int           @id @default(autoincrement())
  bookingTime  DateTime      @default(now())
  customerId   Int
  customer     Customer      @relation(fields: [customerId], references: [id])
  showtimeId   Int
  showtime     Showtime      @relation(fields: [showtimeId], references: [id])
  seatBookings SeatBooking[]
  payments     Payment[]
}

// Junction table for Booking <-> Seat, scoped per Showtime
model SeatBooking {
  id         Int      @id @default(autoincrement())
  bookingId  Int
  booking    Booking  @relation(fields: [bookingId], references: [id])
  seatId     Int
  seat       Seat     @relation(fields: [seatId], references: [id])
  showtimeId Int
  showtime   Showtime @relation(fields: [showtimeId], references: [id])

  @@unique([showtimeId, seatId])   // one seat, one showtime, one booking — ever
}

model Payment {
  id        Int      @id @default(autoincrement())
  amount    Decimal
  status    String   // "paid" | "pending" | "failed"
  paidAt    DateTime?
  bookingId Int
  booking   Booking  @relation(fields: [bookingId], references: [id])
}
```

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


## Reading a Table (Entity Box)

Each box in a diagram = one table. Inside it:

- **PK (Primary Key)** — the column that uniquely identifies each row. No two rows share the same PK.
- **FK (Foreign Key)** — a column that stores another table's PK, used to link two tables together.
- Every other field is just a regular attribute.

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

## Normalization

Normalization is a set of rules for structuring tables so that fixing one fact means updating exactly one row — avoiding **update, delete, and insert anomalies** caused by duplicated data.

**Example problem** — a single unnormalized table:

| student_id | subject | teacher_name | teacher_phone |
|---|---|---|---|
| 1 | Math | Mr. KC | 98765 |
| 1 | Nepali | Mr. KC | 98765 |
| 2 | Math | Mr. KC | 98765 |

If Mr. KC changes his phone number, it has to be updated in every row he appears in — miss one, and the data contradicts itself.

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

Now `Mr. KC`'s phone number exists in exactly one row, regardless of how many subjects he teaches.

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

## Keys & Constraints

**Referential integrity** is the guarantee that a foreign key always points to a row that actually exists — the database rejects any FK value with no matching parent row. What's more interesting is *what happens when the parent row is deleted or updated*, which `onDelete` / `onUpdate` control.

### `onDelete` Behaviors

| Option | Effect when the parent row is deleted |
|---|---|
| `Cascade` | Child rows are deleted automatically along with the parent |
| `Restrict` | Delete is blocked entirely while child rows still reference it |
| `SetNull` | Child's FK column is set to `NULL` — row survives, becomes "orphaned" |
| `NoAction` | No automatic action; delete fails if references exist |

`SetNull` only works if the FK field is nullable (`Int?`) — a required (`Int`) column can never legally hold `NULL`, so Prisma rejects that combination at schema-validation time.

**Choosing the right option is a design decision**, not a default:

```prisma
model Category {
  id       Int       @id @default(autoincrement())
  name     String
  products Product[]
}

model Product {
  id         Int       @id @default(autoincrement())
  name       String
  categoryId Int?
  category   Category? @relation(fields: [categoryId], references: [id], onDelete: SetNull)
}

model Comment {
  id     Int    @id @default(autoincrement())
  text   String
  postId Int
  post   Post   @relation(fields: [postId], references: [id], onDelete: Cascade)
}
```

- `Product → Category` uses `SetNull`: deleting a category shouldn't destroy the products in it — they just become uncategorized.
- `Comment → Post` uses `Cascade`: a comment has no meaning once its post is gone, so it should be deleted with it.
- `Restrict` would fit somewhere like `Order → Customer` if the business rule is "a customer with existing orders can't be deleted until those orders are handled manually."

### `onUpdate`

Same four options, but triggered when the **parent's primary key value itself changes** rather than being deleted. Rarely relevant with auto-increment surrogate keys, since those never change once created — it matters more when a natural key (like an email used as PK) is updated.

### Composite Unique Constraints

```prisma
model Enrollment {
  id        Int @id @default(autoincrement())
  studentId Int
  subjectId Int

  @@unique([studentId, subjectId])
}
```

This does **not** make `studentId` or `subjectId` unique individually — either can repeat many times on their own. It only blocks the exact **pair** from repeating, so the same student can't be enrolled in the same subject twice.

### NOT NULL by Default

Every Prisma field is `NOT NULL` unless marked optional with `?`:

```prisma
name String   // required
bio  String?  // optional
```

Required fields are a form of data integrity enforced by the database itself — once `email` is required, no part of the application ever needs to defensively check for a missing one. Making everything optional "just in case" quietly removes that guarantee and pushes the validation burden into application code instead.

---

## Indexing

An index is a separate, sorted lookup structure the database keeps alongside a table, so it can jump straight to matching rows instead of scanning every row one by one. Primary keys, foreign keys, and `@unique` fields are indexed automatically — anything else needs an explicit `@@index`.

```prisma
model Patient {
  id    Int    @id @default(autoincrement())
  name  String
  phone String

  @@index([phone])
}
```

This makes `WHERE phone = '98765'` fast, since receptionist lookups by phone are frequent but `phone` isn't a key on its own. Indexes speed up reads but add a small cost to every write, so only columns actually used in `WHERE`, `ORDER BY`, or `JOIN` are worth indexing.

---

## ACID

A **transaction** is a group of operations that must all succeed together, or none at all — e.g. a bank transfer, where debiting one account and crediting another only make sense as a single unit.

- **Atomicity** — all steps in a transaction succeed, or none do. If one step fails, everything is rolled back.
- **Consistency** — a transaction can only move the database from one valid state to another, never violating constraints (foreign keys, `NOT NULL`, `@unique`, business rules).
- **Isolation** — concurrent transactions don't interfere with each other's half-finished work; each behaves as if it ran alone.
- **Durability** — once a transaction is committed, it survives permanently, even if the server crashes immediately after.

```prisma
model Account {
  id      Int    @id @default(autoincrement())
  owner   String
  balance Int    @default(0)
}
```

```javascript
async function transferMoney(fromId, toId, amount) {
  return await prisma.$transaction(async (tx) => {
    const sender = await tx.account.findUnique({ where: { id: fromId } });

    if (sender.balance < amount) {
      throw new Error("Insufficient balance");
      // throwing here triggers Atomicity 
    }

    await tx.account.update({
      where: { id: fromId },
      data: { balance: { decrement: amount } },
    });

    await tx.account.update({
      where: { id: toId },
      data: { balance: { increment: amount } },
    });
    // once this resolves, both updates commit together — Durability
    });
}
```

A single Prisma call (e.g. `prisma.user.update()`) is already wrapped in its own implicit transaction. `$transaction` is needed specifically when **multiple separate operations** must succeed or fail together as one unit — like debiting one account and crediting another.

---

## Joins

A **join** combines rows from two tables into one result, matching them on a related column (usually a foreign key pointing to a primary key). Whichever table is named first in the query is the **left table** — LEFT JOIN keeps every row from it, filling `NULL` where there's no match on the other side. INNER JOIN keeps only rows that match on both sides.

**Customer**
| id | name |
|---|---|
| 1 | Ravi |
| 2 | Sita |
| 3 | Hari |

**Order**
| id | customerId | product |
|---|---|---|
| 101 | 1 | Laptop |
| 102 | 2 | Keyboard |

```sql
SELECT Customer.name, Order.product
FROM Customer
LEFT JOIN Order ON Customer.id = Order.customerId;
```

| name | product |
|---|---|
| Ravi | Laptop |
| Sita | Keyboard |
| Hari | NULL |

Hari has no orders but is still kept, since LEFT JOIN never drops rows from the left table.

### Schema

```prisma
model Customer {
  id     Int     @id @default(autoincrement())
  name   String
  orders Order[]
}

model Order {
  id         Int      @id @default(autoincrement())
  product    String
  customerId Int
  customer   Customer @relation(fields: [customerId], references: [id])
}
```

### In Prisma

```javascript
// LEFT JOIN behavior — every customer returned, orders: [] if none
const allCustomers = await prisma.customer.findMany({
  include: { orders: true },
});

// INNER JOIN behavior — only customers with at least one order
const customersWithOrders = await prisma.customer.findMany({
  where: { orders: { some: {} } },
  include: { orders: true },
});
```

Prisma doesn't use `JOIN` keywords directly — `include` generates the join internally and reshapes the result into nested objects/arrays instead of flat rows with `NULL`s. `findMany` always returns an array; each customer object has `orders` nested inside it as its own array, empty if there are no matches.