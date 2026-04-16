---
name: sqlc-jsonb-joins
description: Compose relational JOIN results into Go model types using PostgreSQL to_jsonb() with sqlc. Use whenever a sqlc project has JOIN queries producing flat row structs and you want typed model composition instead — accessing results as row.User.Email, row.Order.Item rather than flat fields. Triggers on: sqlc JOIN queries, sqlc.embed failures, "type explosion" in sqlc, model type reuse, composing relational data in Go, to_jsonb with sqlc, reducing JOIN boilerplate, sqlc column name conflicts.
---

# sqlc jsonb Composition

## Problem

sqlc JOIN queries produce flat structs — every column from every table merged into one type. Each JOIN gets a unique struct, no model reuse, handlers copy fields one-by-one.

`sqlc.embed()` seems like the fix but breaks when tables share column names (`created_at`, `updated_at`, `role`). Go won't compile two embedded structs with duplicate fields.

## Solution

Return one `to_jsonb()` column per joined table. Unmarshal each into the corresponding model type.

## The SQL pattern

```sql
-- name: ListOrdersWithCustomer :many
SELECT
    to_jsonb(o) AS "order",
    to_jsonb(c) AS "customer"
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.status = $1
ORDER BY o.created_at DESC;
```

## What sqlc generates

```go
type ListOrdersWithCustomerRow struct {
    Order    []byte `db:"order" json:"order"`
    Customer []byte `db:"customer" json:"customer"`
}
```

Unmarshal each column into the corresponding model:

```go
var order repo.Order
json.Unmarshal(row.Order, &order)

var customer repo.Customer
json.Unmarshal(row.Customer, &customer)
```

## Why not the alternatives

**`sqlc.embed()`** — breaks when joined tables share column names (`created_at`, `updated_at`). Also can't embed a `nil` struct for nullable joins.

**`jsonb_build_object(...)` combined** — returns one blob, needs an intermediate Go struct.

**Flat rows** — sqlc default. Every JOIN is a unique type with no reuse.

## Three+ tables

One `to_jsonb()` per table. Scales linearly.

```sql
SELECT
    to_jsonb(o)  AS "order",
    to_jsonb(c)  AS "customer",
    to_jsonb(pm) AS "payment"
FROM orders o
JOIN customers c ON c.id = o.customer_id
JOIN payments pm ON pm.order_id = o.id
WHERE o.id = $1;
```

## LEFT JOINs and NULL

`to_jsonb()` on the NULL side produces SQL NULL, which pgx scans as `nil`/empty `[]byte`. `json.Unmarshal` handles this gracefully — returns an error and leaves the target as zero value. Just ignore it:

```go
_ = json.Unmarshal(row.Payment, &payment)
```

## Parent + children in one query

```sql
-- name: GetCustomerWithOrders :one
SELECT
    to_jsonb(c) AS customer,
    COALESCE(jsonb_agg(to_jsonb(o) ORDER BY o.created_at DESC)
        FILTER (WHERE o.id IS NOT NULL), '[]'::jsonb) AS orders
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE c.id = $1
GROUP BY c.id;
```

`COALESCE(..., '[]'::jsonb)` handles zero children. `FILTER (WHERE ...)` excludes the NULL row from LEFT JOIN.

**Gotcha:** `jsonb_agg` with complex expressions sometimes makes sqlc generate `interface{}` instead of `[]byte`. Handle with a type switch at the call site.

## Selective columns

`to_jsonb(t)` serializes every column. Use `jsonb_build_object` to exclude sensitive data or reduce payload size:

```sql
SELECT
    to_jsonb(o) AS "order",
    jsonb_build_object('id', c.id, 'name', c.name, 'email', c.email) AS "customer"
FROM orders o
JOIN customers c ON c.id = o.customer_id
```

