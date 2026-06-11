# 02 — Recursive-descent SQL parser

## Outcome

A hand-written recursive-descent parser that takes a query string and returns
either a typed AST or a parse error pointing at the column in the input. No
external SQL parser dependency.

## Why

TDD §3 and §4.2: the parser strictness decision is the core UX contract.
Unknown SQL constructs must be a parse error with a column-pointing message,
not a permissive best-effort.

## Acceptance criteria

- Supports the documented v1 SQL subset:
  - `SELECT` (column list + `*`), `FROM` (implicit single stream),
    `WHERE`, `GROUP BY`, `ORDER BY` (ASC/DESC), `LIMIT`.
  - Aggregates: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `COUNT(DISTINCT col)`.
  - Scalar functions (pending Open Question 1 sign-off): `LIKE`, `LOWER`,
    `UPPER`, `COALESCE`, `CASE`.
  - Comparison + boolean ops: `=`, `!=`, `<`, `<=`, `>`, `>=`, `AND`, `OR`,
    `NOT`, `IS NULL`, `IS NOT NULL`.
- Any construct outside the documented subset returns a parse error.
- Parse error messages include a 1-based column number into the query string
  and a one-line excerpt with a caret indicator.
- AST node types are exported and have stable field names (consumed by task 04).
- Unit tests cover: each supported construct (happy path), and at least 15
  representative parse-error cases (unknown keyword, missing token, unexpected
  EOF, unbalanced parens, invalid identifier, reserved-word misuse, etc.).

## Out of scope

- Execution (task 04).
- Date functions (deferred to v1.1 per TDD §10).

## Dependencies

01. Open Question 1 must be resolved before merging.
