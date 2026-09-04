# RaceDay – Part 1 Design Notes (README)

This note explains the design decisions behind the ERD and SQL script in
`/docs`, as required where the implementation makes a deliberate choice that
isn't the only "obvious" option.

## 1. One `Users` table, not separate `Organisers` / `Participants` tables

Organisers and Participants share a single `Users` table, distinguished by a
`RoleID` foreign key into a `Roles` lookup table, rather than two separate
tables.

**Why:** the brief specifically asks the plan to "demonstrate understanding
of role-based system design" before any application code is written. A
shared `Users` table with a `RoleID` is the standard way to model
role-based access control (RBAC) — one authentication/identity table, with
role determining what a user is authorised to do at the API layer. It also
avoids duplicating shared columns (name, email, password hash, phone) across
two near-identical tables, and makes it trivial to add a new role later
(e.g. an Admin) without a schema change.

## 2. `Enrolments` as an explicit junction table (not a many-to-many)

A Participant can enter many Categories, and a Category has many Participants
enrolled in it — a many-to-many relationship. Rather than model that as a
plain link table, `Enrolments` is a full entity with its own primary key,
`BibNumber`, `EnrolmentDate`, and `Status`.

**Why:** the relationship carries data of its own (bib number, enrolment
date, confirmed/cancelled status) that doesn't belong to either `Users` or
`Categories`. A composite-key link table couldn't hold this. The
`UQ_Enrolments_ParticipantCategory` constraint still enforces that a
participant can only enrol once per category.

## 3. `Results` as a separate 1:1 entity, not columns on `Enrolments`

Finish time and position are stored in their own `Results` table
(`EnrolmentID` is `NOT NULL UNIQUE`, giving a strict 1:1 relationship) rather
than as extra nullable columns on `Enrolments`.

**Why:** an enrolment exists the moment a participant signs up, long before
the event happens; a result only exists after the event, and only for
participants who finish. Keeping them separate means `Enrolments` isn't full
of NULLs for every future event, and it keeps "who's entered" cleanly
separate from "what happened on race day."

## 4. Feed data assumptions

- Password hashes in the seed script are placeholders
  (`HASHED_PASSWORD_1` etc.) — they will be replaced with real hashed
  values once the Part 2 registration endpoint exists and can generate them
  properly (e.g. bcrypt).
- Bib numbers are prefixed by a short event/city code (`PTA`, `DBN`, `CPT`)
  for readability in sample data; the application layer will define the
  real bib-numbering scheme in Part 2.
- One sample `Result` row is seeded, for the enrolment in the event whose
  date has already passed relative to the other sample events, to show the
  table populated with realistic data without implying every enrolment has
  finished.

## 5. Naming and typing conventions

- All primary keys are `IDENTITY(1,1) INT` surrogate keys, not natural keys
  (e.g. not `Email` as the PK on `Users`), so that display values like email
  or bib number can change without cascading key updates.
- Money fields (`EntryFee`) use `DECIMAL(8,2)`, not `FLOAT`, to avoid
  floating-point rounding errors in currency values.
- Every table has a `CreatedAt`/`RecordedAt` audit column defaulting to
  `GETDATE()`, for basic traceability without needing a full audit log.

No deliberate differences exist between the ERD (`RaceDay_ERD.png` /
`RaceDay_ERD.pdf`) and the SQL script (`RaceDay_Schema.sql`) — the script
implements the ERD exactly as drawn.
