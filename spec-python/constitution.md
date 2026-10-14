# Constitution — Manager Library (Python)

Non-negotiable principles of the project. They are neither style nor preference: every article states **how it is verified**, because a principle that nobody can check is only an intention. A change that violates an article is rejected without discussing the rest.

---

## Article I — The domain depends on nothing

`src/library/domain/` imports no other module of the project and no third-party libraries: only the standard library (`dataclass(frozen=True)`, `datetime`, `typing`); neither SQLAlchemy, nor FastAPI, nor pydantic.

**Why:** it is the only thing that makes it possible to test the invariants without a database, without a server and without a container, and to change the ORM without touching a single rule.
**Verification:** an `import-linter` contract and a test in `tests/architecture/` forbid the domain from depending on `application`, `adapters` or `bootstrap`.

## Article II — The core declares its ports; the world implements them

The application layer only knows the domain. Everything it needs from the outside (repositories, clock, password hashing, cover storage, token issuing) it expresses as a **port that it defines itself**. No framework type and no ORM type ever appears in the signature of a port.

**Verification:** an `import-linter` contract and a test in `tests/architecture/` prevent `application` from importing `adapters`. The application tests run against fakes and never reference `adapters`.

## Article III — A single composition root

One single module (`src/library/bootstrap/composition/port_bindings.py`) is where a port meets its adapter. Outside it, no infrastructure is ever instantiated.
**Verification:** no instantiation of an `adapters` class outside that module; checked with an architecture test.

## Article IV — SOLID is not decorative

| Principle | Concrete application | Verification |
|---|---|---|
| **SRP** | One application service = one use case. `LendBooksService` lends and does nothing else | A service with two reasons to change gets split |
| **OCP** | Changing the cover storage (disk → S3) touches neither domain nor application | The new adapter only adds one line in the composition |
| **LSP** | Test fakes replace the real adapter without changing expectations | The same test passes against the fake and against the real one |
| **ISP** | Small ports: `Clock`, `PasswordHasher`, `TokenIssuer`, `CoverStorage` | No implementer is left with unused methods |
| **DIP** | The port is defined by whoever consumes it | The ports live in the application package |

## Article V — The schema belongs to the migrations

The `infra` repository brings up an **empty** engine. All the DDL (tables, indexes, constraints, extensions and seed data) lives in the application migrations (Alembic revisions). See [ADR-001](adr/adr-001-schema-ownership.md).
**Verification:** `infra` contains no `CREATE TABLE`.

## Article VI — Whatever the database can enforce, the database enforces

A rule that lives only in code protects the application, not the data. Every invariant expressible in the engine (`CHECK`, `UNIQUE`, FK) is declared there. Whatever is not enforced is marked **pending** in [`data-model.md`](data-model.md); nothing is promised.
**Verification:** the queries in `data-model.md` §10 match what the document declares.

## Article VII — Derived values are computed, not stored

The state of a loan (`open`, `overdue`, `closed`) and the days of delay are computed. There is no column for them.
**Verification:** `data-model.md` lists no `status` column and no `days_late` column.

## Article VIII — Frozen values do not follow the catalogue

The loan line stores the title, the ISBN and the category as they were at the instant of the loan. No report ever reads them from the live catalogue.
**Verification:** a test that renames a book after it was lent and checks that the report keeps the old title ([ADR-004](adr/adr-004-frozen-aggregated-report.md)).

## Article IX — No secret ever enters the repository or a log

Neither passwords, nor precomputed hashes, nor the token key. Every secret comes from the environment. The real `.env` is in `.gitignore`; only `.env.example` is versioned.
**Verification:** `gitleaks` or a pattern `grep` in CI; no HTTP response includes `password` nor `password_hash`.

## Article X — The engine wins

If the data document and the database disagree, the document is broken and the document gets fixed.

## Article XI — Language

**Everything is written in English**: code, columns and routes, all documentation, all identifiers, and every message returned to the user. There is no Spanish in this project.

## Article XII — Nothing is taken as done without evidence

A task in [`tasks.md`](tasks.md) is finished when its **evidence** exists and can be executed, not when it is "already implemented".