# Data model — Manager Library

**The only place where the data model lives.** Whoever implements it does not need to open the code nor connect to the engine to know what is there, of what type and with what rule.

- Reference engine: PostgreSQL 16. Schema `library`. Server in UTC.
- Governed by [`constitution.md`](constitution.md) and [`spec.md`](spec.md). If something contradicts this document, this document wins.

Every rule carries one of **three marks**: **engine** (the database enforces it), **domain only** (only the code does), **pending (T-xx)** (the corresponding task in [`tasks.md`](tasks.md) adds it).

---

## 0. Naming convention

| Element | Convention | Example |
|---|---|---|
| Table | `snake_case`, English, **singular** | `loan_item` |
| Column | `snake_case`, English, singular, ASCII | `due_at` |
| Hand-written constraint | `ck_{table}_{rule}` | `ck_book_available_range` |
| Schema | `library` | `library.loan` |

`user` is written `library."user"` in raw SQL; the ORM quotes it on its own.

## 1. Glossary

| Term | Definition | Where it lives |
|---|---|---|
| **Book** | Catalogue title, with author, ISBN, category, copies and optional cover | `book` |
| **Category** | Fixed classification of five, seeded, without maintenance | `category` |
| **Copy** | A physical copy of a book. It is counted, not identified | `book.total_copies`, `book.available_copies` |
| **Member** | A person who borrows. Does not log in | `member` |
| **Loan** | A consummated fact: who asked for it, who registered it and when | `loan` |
| **Loan line** | A book inside a loan, with its due date and its return | `loan_item` |
| **Loan state** | `open`, `overdue` or `closed`. **Computed, not stored** | no column |
| **User** | Internal operator with the role `admin` or `librarian` | `user` |
| **Frozen** | Copy of the value at the instant of the loan; it does not follow the catalogue | `loan_item.book_title`, `isbn`, `category_name` |

## 2. Entities and relationships

```mermaid
erDiagram
    category ||--o{ book      : "classifies"
    member   ||--o{ loan      : "asks for"
    user     ||--o{ loan      : "registers"
    loan     ||--|{ loan_item : "composes"
    book     ||--o{ loan_item : "lent in (FK RESTRICT)"
```

Six entities, six tables. There is no report table, no fines table and no audit table.

## 3. Physical model

No column carries a `DEFAULT`: the values are set by the domain.

**`category`** — read-only seed.

| Column | Type | Nullable | Note |
|---|---|---|---|
| `id` | `uuid` | no | PK. Literal identifiers (see §9) |
| `name` | `varchar(120)` | no | Unique |

**`book`**

| Column | Type | Nullable | Note |
|---|---|---|---|
| `id` | `uuid` | no | PK |
| `title` | `varchar(250)` | no | Trimmed by the domain |
| `author` | `varchar(200)` | no | |
| `isbn` | `char(13)` | no | Digits only. Unique among active books (partial index) |
| `category_id` | `uuid` | no | Restrictive FK to `category` |
| `total_copies` | `integer` | no | ≥ 1 |
| `available_copies` | `integer` | no | Between 0 and `total_copies` |
| `cover_key` | `varchar(512)` | **yes** | Opaque key; never a path nor bytes; never an empty string |
| `deleted_at` | `timestamptz` | yes | Soft delete. Null = active |

**`member`**

| Column | Type | Nullable | Note |
|---|---|---|---|
| `id` | `uuid` | no | PK |
| `full_name` | `varchar(200)` | no | |
| `document_number` | `varchar(30)` | no | Unique |

**`loan`**

| Column | Type | Nullable | Note |
|---|---|---|---|
| `id` | `uuid` | no | PK |
| `member_id` | `uuid` | no | Restrictive FK to `member` |
| `loaned_by_user_id` | `uuid` | no | Restrictive FK to `user` |
| `loaned_at` | `timestamptz` | no | Instant of the loan |

**`loan_item`**

| Column | Type | Nullable | Note |
|---|---|---|---|
| `id` | `uuid` | no | PK |
| `loan_id` | `uuid` | no | FK with `ON DELETE CASCADE` |
| `book_id` | `uuid` | no | Restrictive FK to `book` |
| `book_title` | `varchar(250)` | no | Frozen; same width as `book.title` |
| `isbn` | `char(13)` | no | Frozen |
| `category_name` | `varchar(120)` | no | Frozen; **no FK on purpose** |
| `due_at` | `timestamptz` | no | `loaned_at` + 14 days, computed by the domain |
| `returned_at` | `timestamptz` | **yes** | Null while it is not returned |

**`user`**

| Column | Type | Nullable | Note |
|---|---|---|---|
| `id` | `uuid` | no | PK |
| `username` | `varchar(120)` | no | Unique, lowercase, trimmed |
| `password_hash` | `varchar(512)` | no | Never indexed |
| `role` | `varchar(40)` | no | `admin` or `librarian` |

Cross-cutting rules: **all timestamps are `timestamptz`**; **there are no audit columns** (`created_at`/`updated_at`) because no requirement asks for them.

## 4. Constraints and indexes: where each rule lives

| Rule | Object | Mark |
|---|---|---|
| Primary keys of the six tables | `PK_*` | **engine** |
| `category.name` unique | unique index | **engine** |
| `member.document_number` unique | unique index | **engine** |
| `user.username` unique | unique index | **engine** |
| `isbn` unique among active books | partial unique index `WHERE deleted_at IS NULL` | **engine** |
| `book.category_id` → `category.id` RESTRICT | FK | **engine** |
| `loan.member_id` → `member.id` RESTRICT | FK | **engine** |
| `loan.loaned_by_user_id` → `user.id` RESTRICT | FK | **engine** |
| `loan_item.loan_id` → `loan.id` CASCADE | FK | **engine** |
| `loan_item.book_id` → `book.id` RESTRICT | FK | **engine** |
| `0 <= available_copies <= total_copies` | `ck_book_available_range` | **engine** — last barrier of concurrency |
| `total_copies >= 1` | `ck_book_total_positive` | **pending (T-18)** · today domain only |
| Unique `(loan_id, book_id)` | unique index | **engine** |
| `role` in (`admin`,`librarian`) | `ck_user_role` | **pending (T-18)** · today domain only |
| Maximum 3 not returned per member | — | **domain only** (not expressible in a `CHECK`) |
| Member with overdue loans does not borrow | — | **domain only** |
| Subtracting a copy and creating the line are one operation | — | **domain only** (one transaction) |
| A loan has 1 to 3 lines | — | **domain only** |
| Loans and returns immutable except `returned_at` | — | **domain only** (by absence of operation) |

Access indexes: `book (category_id, title)` partial over active books; `loan (loaned_at)`; `loan_item (book_id)`; `loan_item (due_at) WHERE returned_at IS NULL` for overdue.

## 5. Foreign key policy

| # | FK | `ON DELETE` | Why |
|---|---|---|---|
| FK-1 | `book.category_id` | RESTRICT | A category with books is not deleted |
| FK-2 | `loan_item.loan_id` | CASCADE | Composition: the line does not live without its loan (never fired: loans are not deleted) |
| FK-3 | `loan_item.book_id` | RESTRICT | Last-resort barrier: a manual delete cannot orphan the history |
| FK-4 | `loan.member_id` | RESTRICT | The member of a loan does not disappear |
| FK-5 | `loan.loaned_by_user_id` | RESTRICT | Authorship is accounting data |

`ON UPDATE NO ACTION` on all of them: the keys are application-generated UUIDs and they do not change.

## 6. Access patterns

| # | Pattern | Frequency |
|---|---|---|
| Q1 | Search books (text, category, active, ordered by title, paginated) | High |
| Q2 | Book by id | High |
| Q3 | Books by batch of ids (before lending) | High, contention point |
| Q4 | List categories | High |
| Q5 | Loan with its lines | Medium |
| Q6 | Loans by range, state and member | High |
| Q7 | Lines not returned of a member (count and overdue) | High, on every loan |
| Q8 | Global overdue | Medium |
| Q9 | Report: most borrowed by range | High, the most expensive |
| Q10 | User by username | High, on every login |

## 7. Privacy and retention

| Table | Attribute | Classification | Handling |
|---|---|---|---|
| `user` | `username` | Personal data | Restricted access |
| `user` | `password_hash` | **Secret** | Never in logs nor responses; never indexed |
| `member` | `full_name`, `document_number` | **Personal data** | Only `admin` and `librarian`; **never** in the report nor in public endpoints |
| `loan`, `loan_item` | all | Commercial; `member_id` is indirect personal data | Indefinite retention |
| `book`, `category` | all | Public | Soft delete / no deletion |

The report aggregates by book: **it does not expose members nor librarians**.

**Retention:** loans and lines are never deleted; books only have a soft delete; the cover binary is the **only** data that is physically deleted. Mandatory order: first `cover_key` is nulled and the transaction is committed; then the binary is deleted.

## 8. Audit

The system **does not** keep `created_at`/`updated_at`. The business instants already have a column: `loan.loaned_at`, `loan_item.returned_at` and `book.deleted_at`. If a question like "who changed this title?" shows up, the decision goes back to the owner.

## 9. Seed strategy

The **five categories** go in the initial migration, with fixed identifiers:

| `id` | `name` |
|---|---|
| `11111111-1111-4111-8111-111111111111` | Fiction |
| `22222222-2222-4222-8222-222222222222` | Science |
| `33333333-3333-4333-8333-333333333333` | Technology |
| `44444444-4444-4444-8444-444444444444` | History |
| `55555555-5555-4555-8555-555555555555` | Children |

The **initial administrator is not seeded by the database**: the application boot creates it from environment credentials, because only the hashing port can produce the hash. The demo data seeder (`tool`) uses the **API**, never direct SQL.

## 10. How this document is checked for lying

```sql
-- Must return the columns of §3, with no DEFAULT at all
SELECT table_name, ordinal_position, column_name, data_type, is_nullable, column_default
FROM information_schema.columns WHERE table_schema = 'library' ORDER BY 1, 2;

-- Constraints: PK, FK and CHECK
SELECT c.conrelid::regclass, c.conname, c.contype, pg_get_constraintdef(c.oid)
FROM pg_constraint c JOIN pg_namespace n ON n.oid = c.connamespace
WHERE n.nspname = 'library' ORDER BY 1, 3, 2;

-- Indexes
SELECT tablename, indexname, indexdef FROM pg_indexes WHERE schemaname = 'library' ORDER BY 1, 2;
```

If a rule marked **engine** does not appear, somebody deleted it. If one marked **domain only** appears, it has been pushed down and has to be reclassified.
The engine wins: the document is broken and the document gets fixed.