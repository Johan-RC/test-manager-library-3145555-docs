# API contract — Manager Library

The HTTP contract is **language-independent**: whoever implements it in React + Laravel must respect it exactly as whoever implements it in Python or .NET. What each operation does is in [`spec.md`](spec.md); what is stored is in [`data-model.md`](data-model.md).

## 1. Conventions

| Topic | Rule |
|---|---|
| Base | `/api/v1` |
| Format | JSON, UTF-8. Field names in `snake_case` |
| Identifiers | UUID v4 as text |
| Dates | ISO 8601 in UTC, with `Z` (`2026-10-14T14:15:00Z`) |
| Money | None. The system does not handle amounts |
| Authentication | `Authorization: Bearer <token>` everywhere except `POST /auth/login` and `GET /health` |
| Pagination | `?page=1&size=20`. `size` max 50. The response is `{ "items": [...], "page": 1, "size": 20, "total": 134 }` |
| Language | All text is in English: error messages, field names and codes |

### Error body (the only one, for every error)

```json
{
  "code": "BOOK_NOT_AVAILABLE",
  "message": "No available copies left of \"One Hundred Years of Solitude\".",
  "details": [ { "field": "items[0].book_id", "issue": "no copies available" } ]
}
```

`details` may be empty. It **never** includes stack traces, SQL, server paths or the password.

### Status codes

| Status | When |
|---|---|
| 200 / 201 / 204 | Success / created / no content |
| 401 | No token, invalid or expired token, invalid credentials |
| 403 | Insufficient role |
| 404 | Does not exist (or has been soft deleted) |
| 409 | State conflict: duplicate, no copies available, already returned, member with overdue loans, book with open loans |
| 422 | Validation: invalid field, broken input rule, inverted date range |

## 2. Business error codes

| `code` | Status | Origin |
|---|---|---|
| `INVALID_CREDENTIALS` | 401 | CA-01.2 |
| `FORBIDDEN_ROLE` | 403 | CA-01.3 |
| `NOT_FOUND` | 404 | Any resource |
| `USERNAME_TAKEN` | 409 | CA-02.2 |
| `ISBN_TAKEN` | 409 | CA-03.2 |
| `DOCUMENT_TAKEN` | 409 | CA-05.1 |
| `BOOK_HAS_OPEN_LOANS` | 409 | CA-03.5 |
| `BOOK_NOT_AVAILABLE` | 409 | CA-06.3 |
| `MEMBER_HAS_OVERDUE` | 409 | CA-06.5 |
| `ITEM_ALREADY_RETURNED` | 409 | CA-07.2 |
| `MEMBER_LIMIT_EXCEEDED` | 422 | CA-06.4 |
| `DUPLICATE_BOOK_IN_LOAN` | 422 | CA-06.2 |
| `VALIDATION_ERROR` | 422 | Any invalid field |
| `INVALID_DATE_RANGE` | 422 | CA-10.2 |

## 3. Endpoints

### 3.1 Authentication and health

| Method | Path | Role | Description |
|---|---|---|---|
| POST | `/auth/login` | public | Returns a token |
| GET | `/health` | public | `{ "status": "ok" }` |

`POST /auth/login`

```json
// request
{ "username": "ana", "password": "********" }
// 200
{ "access_token": "eyJ...", "token_type": "Bearer", "expires_in": 28800, "role": "librarian" }
```

### 3.2 Users

| Method | Path | Role | Description |
|---|---|---|---|
| POST | `/users` | admin | Creates a librarian |

```json
// request  (role only accepts "librarian")
{ "username": "Ana ", "password": "********", "role": "librarian" }
// 201  (without password nor password_hash)
{ "id": "…", "username": "ana", "role": "librarian" }
```

### 3.3 Categories

| Method | Path | Role | Description |
|---|---|---|---|
| GET | `/categories` | authenticated | The five categories, ordered by name |

### 3.4 Books

| Method | Path | Role | Description |
|---|---|---|---|
| GET | `/books` | authenticated | Search: `q`, `category_id`, `page`, `size` |
| GET | `/books/{id}` | authenticated | Detail |
| POST | `/books` | admin | Creates |
| PUT | `/books/{id}` | admin | Replaces title, author, category and total |
| DELETE | `/books/{id}` | admin | Soft delete |
| PUT | `/books/{id}/cover` | admin | Uploads a cover (`multipart/form-data`, field `file`) |
| DELETE | `/books/{id}/cover` | admin | Removes the cover |

Book (response):

```json
{
  "id": "…", "title": "One Hundred Years of Solitude", "author": "Gabriel García Márquez",
  "isbn": "9780307474728", "category": { "id": "…", "name": "Novel" },
  "total_copies": 4, "available_copies": 3, "cover_key": null
}
```

`POST /books` (request): `title`, `author`, `isbn`, `category_id`, `total_copies`. The server computes the available copies.
`cover_key` is an opaque key: to render the cover, `GET /books/{id}/cover` is used, which returns the binary or 404.

### 3.5 Members

| Method | Path | Role | Description |
|---|---|---|---|
| GET | `/members` | admin, librarian | List and search: `q`, `page`, `size` |
| GET | `/members/{id}` | admin, librarian | Detail with `active_loans` (count of items not returned) |
| POST | `/members` | admin, librarian | Creates: `full_name`, `document_number` |

### 3.6 Loans

| Method | Path | Role | Description |
|---|---|---|---|
| POST | `/loans` | admin, librarian | Registers a loan |
| GET | `/loans` | admin, librarian | List: `from`, `to`, `status`, `member_id`, `page`, `size` |
| GET | `/loans/{id}` | admin, librarian | Detail with lines and state |
| POST | `/loans/{loan_id}/items/{item_id}/return` | admin, librarian | Returns one line |
| GET | `/loans/overdue` | admin, librarian | Overdue lines |

`POST /loans`

```json
// request  (1 to 3 elements, no repeated books)
{ "member_id": "…", "items": [ { "book_id": "…" }, { "book_id": "…" } ] }
// 201
{
  "id": "…", "member_id": "…", "loaned_at": "2026-10-14T14:15:00Z", "status": "open",
  "items": [
    { "id": "…", "book_id": "…", "book_title": "One Hundred Years of Solitude", "isbn": "9780307474728",
      "category_name": "Novel", "due_at": "2026-10-28T14:15:00Z", "returned_at": null }
  ]
}
```

`status` is computed: `open` (there are lines not returned and none overdue), `overdue` (some line not returned and overdue), `closed` (all returned).

`POST /loans/{loan_id}/items/{item_id}/return` → 200 with the line (`returned_at` filled).

`GET /loans/overdue` → `items`: `{ "loan_id", "item_id", "book_title", "member_name", "due_at", "days_late" }`. Here the member name is shown, because this is an operational listing; **the report in §3.7 does not show it**.

### 3.7 Report

| Method | Path | Role | Description |
|---|---|---|---|
| GET | `/reports/most-borrowed` | admin | `from` and `to` are mandatory (date, both inclusive) |

```json
{
  "from": "2026-10-01", "to": "2026-10-31",
  "rows": [
    { "book_id": "…", "book_title": "One Hundred Years of Solitude", "category_name": "Novel", "times_borrowed": 12 }
  ]
}
```

One row per frozen `book_id + book_title + category_name`. Order: `times_borrowed` descending, then title. No members and no librarians.

## 4. Security

| Aspect | Rule |
|---|---|
| Token | Signed JWT, 8 hours, with `sub` and `role`. No personal data inside |
| Passwords | Only the hash is stored; the hash never leaves in any response |
| Authorization | Evaluated on every endpoint by role; a 403 does not reveal whether the resource exists |
| CORS | Only the frontend origin configured by the environment |
| Cover upload | Only `image/jpeg` and `image/png`, max 2 MB; the type is validated by content, not by extension |