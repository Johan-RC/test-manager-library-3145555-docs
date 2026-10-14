# Specification — Manager Library

**What is built and why.** A system for a small library to keep track of its book catalogue, its members and its loans, and to see which books are lent the most.

Governed by [`constitution.md`](constitution.md). The data model lives only in [`data-model.md`](data-model.md) and the HTTP contract only in [`api-contract.md`](api-contract.md).

---

## 1. Actors

| Actor | What they do | How they authenticate |
|---|---|---|
| **Administrator** (`admin`) | Maintains the book catalogue and the members; creates librarians; sees every report | Username and password → token |
| **Librarian** (`librarian`) | Registers loans and returns; queries the catalogue, members and overdue loans | Username and password → token |
| **Member** | A person who borrows books. **Does not log in**: they are data, not a user | — |

The `admin` role is provisioned by the deployment from environment variables. **Nobody grants it at runtime.**

## 2. Scope

**Inside:** book catalogue with optional cover, five fixed categories, members, loans with several lines, per-line return, overdue books, most-borrowed report, login with roles.

**Outside, and it is not reopened:** fines and payments, reservations, renewals, email or SMS, bulk import, loans between branches, book attributes beyond those of §3 (no publisher, no year, no language).

## 3. User stories and acceptance criteria

### HU-01 · Log in
As a user I want to authenticate myself in order to operate according to my role.
- **CA-01.1** With valid credentials a token with an 8-hour validity is returned.
- **CA-01.2** With invalid credentials it responds 401 with a message that **does not** say whether the user or the password failed.
- **CA-01.3** Every endpoint except login and `/health` requires a token. Without a token: 401. Insufficient role: 403.

### HU-02 · Create librarians (admin)
- **CA-02.1** The admin creates a user with a username, a password and the role `librarian`.
- **CA-02.2** The username is stored lowercased and trimmed; it is unique. Repeated: 409.
- **CA-02.3** Requesting the `admin` role through the API responds 422. The password is never returned nor logged.

### HU-03 · Manage books (admin)
- **CA-03.1** Create a book with title, author, ISBN, category and total copies. The available copies start equal to the total.
- **CA-03.2** The ISBN has 13 digits and is unique among active books. Repeated: 409.
- **CA-03.3** Total copies ≥ 1. Changing the total adjusts the available copies by the same difference and **never** leaves available copies < 0 (422).
- **CA-03.4** Uploading a cover (JPG/PNG, max 2 MB) stores an **opaque key**; without a cover the field is `null`, never an empty string.
- **CA-03.5** Deleting a book is a **soft delete**: it stops showing in the catalogue but it stays in past loans. A book with open loans cannot be deleted (409).

### HU-04 · Search books
- **CA-04.1** Partial text search on title or author, filter by category, active books only, ordered by title, paginated (`page`, `size`, max 50).
- **CA-04.2** The response includes the total number of results.
- **CA-04.3** Requesting a book by a non-existent or deleted id: 404.

### HU-05 · Manage members
- **CA-05.1** The admin and the librarian create members with a name and a document. The document is unique: repeated 409.
- **CA-05.2** They are listed and searchable by name or document, paginated.
- **CA-05.3** Members are never deleted.

### HU-06 · Register a loan
- **CA-06.1** A loan belongs to **one** member, it is registered by the authenticated user and it carries from 1 to 3 lines.
- **CA-06.2** The same book is not repeated within a loan (422).
- **CA-06.3** Each line subtracts **1** available copy in the same operation. With no copies available: 409 and **nothing is registered**.
- **CA-06.4** A member cannot have more than **3 books not returned** in total (422 with the current count).
- **CA-06.5** A member with any **overdue** book cannot request another one (409).
- **CA-06.6** The due date is `loaned_at + 14 days`. The line freezes the title, the ISBN and the category name **at that instant**.
- **CA-06.7** A registered loan is never edited nor deleted.

### HU-07 · Register a return
- **CA-07.1** **One line** is returned at a time. It sets `returned_at` and adds 1 to the available copies.
- **CA-07.2** Returning an already returned line: 409. It **never** adds twice.
- **CA-07.3** Available copies never exceed the total (the database guarantees it).
- **CA-07.4** When all lines are returned, the loan shows as **closed**.

### HU-08 · View a loan and list loans
- **CA-08.1** Requesting a loan returns its lines and its computed state (`open`, `overdue`, `closed`).
- **CA-08.2** Listing by date range, state and member, ordered by date descending, paginated.
- **CA-08.3** The state **is computed**, not stored.

### HU-09 · Overdue books
- **CA-09.1** Lists the lines not returned whose `due_at` is before now: book, member, days late.
- **CA-09.2** Ordered by most days late first. Only `admin` and `librarian`.

### HU-10 · Most-borrowed books report
- **CA-10.1** Given a date range, one row per **book and frozen category name**, with the number of times it was lent, ordered from highest to lowest.
- **CA-10.2** End before start: 422. Range with no loans: empty list, not an error.
- **CA-10.3** The report **is not broken down** by librarian nor by member.
- **CA-10.4** A report for a closed period **does not change** even if the book or its category is later renamed.
- **CA-10.5** Only `admin`.

## 4. Business rules

| # | Rule |
|---|---|
| RN-1 | Available copies are between 0 and the total, always. |
| RN-2 | Loan period: 14 days. Maximum 3 books not returned per member. |
| RN-3 | A member with overdue books does not borrow more until returning them. |
| RN-4 | What is frozen in the line (title, ISBN, category) does not follow the catalogue. |
| RN-5 | Loans and returns are never deleted. Books only have a soft delete. |
| RN-6 | Dates are UTC instants. All text — documentation, code, identifiers and user-facing messages — is in English. |

## 5. Non-functional requirements

| # | Requirement | How it is measured |
|---|---|---|
| RNF-1 | The book search responds in less than 300 ms with 5 000 books | Load test with the `tool` seeder |
| RNF-2 | Two librarians lending the **last copy** at the same time: one wins, the other gets 409 | Concurrency test |
| RNF-3 | The password never travels nor is stored in clear text; it never appears in logs | Review of logs and responses |
| RNF-4 | Every HTTP error uses the same body (see the contract) | Contract test |
| RNF-5 | The system starts with a single command and the database starts empty | `README` of `infra` |