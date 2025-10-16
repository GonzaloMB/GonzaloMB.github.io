---
title: "Clean Code SAP CAP"
date: 2025-10-16
author: GonzaloMB
tags: ["sap", "cap", "cds", "nodejs", "odata"]
---

Practical guidelines to structure CAP projects for clarity, safety, and evolvability. Focus on DB (CDS), SRV (services), projections, actions/functions, handlers, and annotations, with concise CDS/JS examples.

---

## Core Principles

- Separation of concerns: db schema ≠ service contract ≠ UI annotations.
- Projection‑first: expose stable service projections, not raw db entities.
- Explicit constraints: types, validations, and invariants close to the data.
- Safe mutations: transactional handlers, idempotency, and clear errors.
- Versioning & evolution: additive changes, deprecate before removal.

---

## DB Layer (db/\*.cds)

Use a namespace, clear names, strong types, and constraints in the model.

```abap
namespace app.domain;

type PositiveInt : Integer @assert.range: [0, 999999];
type NonEmptyString : String(255) @assert.notEmpty;

entity Book {
  key ID     : UUID;
      title  : NonEmptyString @Common.Label: 'Title';
      stock  : PositiveInt     @Common.Label: 'Stock';
      author : Association to Author;
      createdAt : Timestamp @cds.on.insert: $now;
      updatedAt : Timestamp @cds.on.update: $now;
}

entity Author {
  key ID   : UUID;
      name : NonEmptyString;
  books    : Composition of many Book on books.author = $self;
}
```

Tips

- Types encode invariants (`PositiveInt`, `NonEmptyString`).
- Use `UUID` for keys; avoid overloading natural keys.
- Prefer `Composition` for lifecycle ownership; `Association` for references.
- Auditing fields are set with `@cds.on.insert/update`.

---

## Service Layer (srv/\*.cds)

Project DB entities into a service contract. Hide internals; keep naming stable.

```abap
using app.domain as db from '../db/schema';

service CatalogService @(path: '/catalog') {
  entity Books as projection on db.Book
    excluding { createdAt, updatedAt };

  entity Authors as projection on db.Author {
    ID, name
  };

  action Restock(bookID: UUID, amount: Integer) returns Integer;
  function StockOf(bookID: UUID) returns Integer;
}
```

Tips

- Projections allow renames, hides, and joins without breaking DB.
- Keep actions for commands (state changes) and functions for queries.
- Add `@(path: ...)` to control routing explicitly.

---

## Annotations (app/\*/annotations.cds)

Keep UI and behavior annotations out of DB/SRV files.

```abap
using CatalogService as service from '../../srv/catalog-service';

annotate service.Books with @(
  UI.SelectionFields: [ title ],
  UI.LineItem: [
    { $Type: 'UI.DataField', Value: title },
    { $Type: 'UI.DataField', Value: stock }
  ]
);
```

---

## Security & Authorization

Use `@restrict` to define who can do what. Prefer roles and deny by default.

```abap
@requires: 'authenticated-user'
service CatalogService {
  @restrict: [{ grant: 'READ', to: ['viewer','admin'] }]
  entity Books as projection on db.Book;

  @restrict: [{ grant: 'WRITE', to: ['admin'] }]
  action Restock(bookID: UUID, amount: Integer) returns Integer;
}
```

Add tenant/ownership filters with `@restrict.where: 'author_ID = $user.id'` when appropriate.

---

## Handlers (srv/\*.js)

Keep handlers cohesive, small, and transactional. Validate inputs early, return clear errors, and prefer repository‑like access.

```js
const cds = require("@sap/cds");

module.exports = cds.service.impl(function () {
  const { Book } = this.entities;

  this.on("Restock", async (req) => {
    const { bookID, amount } = req.data;
    if (!amount || amount <= 0) return req.error(400, "amount must be > 0");

    return cds.tx(req).run(async (tx) => {
      const [book] = await tx.run(SELECT.one.from(Book).where({ ID: bookID }));
      if (!book) return req.error(404, "Book not found");

      const newStock = (book.stock ?? 0) + amount;
      await tx.run(UPDATE(Book).set({ stock: newStock }).where({ ID: bookID }));
      return newStock;
    });
  });

  this.on("StockOf", async (req) => {
    const { bookID } = req.data;
    const row = await SELECT.one`stock`.from(Book).where({ ID: bookID });
    if (!row) return req.error(404, "Book not found");
    return row.stock ?? 0;
  });
});
```

Tips

- Use `cds.tx(req)` for atomic sequences.
- Validate the contract (presence, ranges) before DB work.
- Return typed values (number, object) rather than strings.

---

## Input Validation: Where?

Layered validation is OK; prefer closest to data.

- CDS types/annotations for structural constraints (lengths, ranges).
- Service actions for business rules (e.g., idempotency, cross‑entity checks).
- Optional UI validation for UX, not as a security barrier.

Example (CDS range + handler rule):

```cds
type RestockAmount : Integer @assert.range: [1, 10000];
```

```js
if (amount > 1000 && !req.user.is("admin")) req.error(403, "Limit exceeded");
```

---

## Queries & Projections

Prefer `SELECT` with explicit fields. Avoid `SELECT *` in handlers to control data shape and avoid coupling.

```js
const book = await SELECT.one
  .from(Book, (b) => {
    b.ID, b.title, b.stock;
  })
  .where({ ID });
```

Expose read models via projection entities (SRV) instead of custom SQL in handlers whenever possible.

---

## Actions vs. Functions

- Use actions (commands) for state changes, side effects, or long‑running jobs.
- Use functions (queries) when the outcome is pure and idempotent.
- For bulk operations, prefer arrays in a single action call; handle per‑item errors with aggregated responses.

Bulk input type example:

```abap
type BooksInput { title: String; stock: Integer; }
action UploadBooks(items: array of BooksInput) returns Integer;
```

---

## Errors & Problem Details

Return meaningful codes/messages. Avoid leaking internals.

```js
if (!req.data.title) return req.error(400, "title is required");
if (!req.user.is("admin")) return req.reject(403, "forbidden");
```

For batch operations, collect item errors and return a summary structure (with indices and messages).

---

## Configuration & Features

`package.json` snippets that improve developer experience:

```json
{
  "cds": {
    "requires": { "db": "sqlite" },
    "features": { "odata_new_parser": true }
  }
}
```

Use the new OData parser for advanced `$apply` pipelines when using FE v4.

---

## Testing (cds.test)

Keep handlers small and test them with cds test utilities or supertest against a running server.

```js
// example pseudo‑test
const srv = await cds.serve("CatalogService").in(app);
await POST("/catalog/Restock", { bookID, amount: 5 }).expect(200);
```

Mock external dependencies (e.g., messaging, destinations) behind small modules.

---

## Performance Tips

- Avoid N+1 queries: batch with `IN` clauses or joins in projections.
- Use indexes on filterable fields; prefer UUID over long varchar keys.
- Paginate consistently (`$top`, `$skip`, stable order by key).
- Consider server‑side aggregates via annotations and `$apply` instead of client loops.

---

## Checklist (TL;DR)

- DB: types, constraints, compositions, auditing.
- SRV: projections, stable names, actions for commands, functions for queries.
- Annotations in app layer, not DB/SRV.
- Handlers: transactional, validate early, clear errors, explicit queries.
- Security: `@restrict` with roles; deny by default.
- Tests for handlers and contracts; mock externals.
- Plan for evolution: projections enable painless renames/hiding.

---
