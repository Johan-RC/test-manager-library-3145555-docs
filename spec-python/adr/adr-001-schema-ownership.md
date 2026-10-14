# ADR-001 · The schema belongs to the application migrations

- **Status:** accepted
- **Context:** the `infra` repository brings up a database engine. Somebody could put the DDL in a container start-up script and another copy in the application migrations; with two owners nobody knows which schema runs in production.
- **Decision:** **all the DDL lives in the application migrations** (tables, indexes, constraints and the category seed). `infra` only creates the empty engine, the user and the volume.
- **Rejected alternatives:** SQL scripts in `infra/db/init/` (a second source of truth); ORM in "create tables on boot" mode (it leaves no history and no reverse).
- **Consequences:** a PostgreSQL extension or a new index is created in the **same** migration that uses it. A schema changed by hand in the engine is a defect, not a variant.
- **How it is verified:** `infra` contains no `CREATE TABLE`; the queries in `data-model.md` §10 match the migrations.