# ADR-003 · Books are soft-deleted, not erased

- **Status:** accepted
- **Context:** a loan line points to a book. If the book were physically deleted, the history and the report would be broken.
- **Decision:** the book carries `deleted_at`. Deleting fills it; the catalogue filters on `deleted_at IS NULL` in a single place (a global filter). A book with open loans **cannot** be deleted (409 `BOOK_HAS_OPEN_LOANS`). The uniqueness of the ISBN is a **partial** index over active books, so a deleted ISBN can be registered again.
- **Rejected alternatives:** physical deletion with cascade (it destroys the history); marking `active = false` (it does not say when).
- **Consequences:** `loan_item.book_id` carries a `RESTRICT` FK as a **last-resort barrier**: a manual `DELETE` or a future code change fails loudly instead of corrupting the history. The cover binary is the only data that is physically deleted.
- **How it is verified:** requesting a soft-deleted book returns 404; a `DELETE` on a book with lines fails because of the FK.