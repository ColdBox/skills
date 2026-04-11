---
name: coldbox-orm-qb
description: "Use this skill when building SQL queries with the QB query builder in ColdBox, using the fluent where/join/orderBy/groupBy/having API, constructing complex subqueries, performing aggregate functions (count/sum/avg/min/max), inserting/updating/deleting records with QB, or using raw expressions."
---

# QB Query Builder in ColdBox

## Overview

QB is a fluent, database-agnostic query builder for CFML/BoxLang. It provides an expressive chainable API for building SQL queries without writing raw SQL, supporting MySQL, PostgreSQL, MSSQL, and Oracle grammars.

## Installation & Configuration

```bash
box install qb
```

```boxlang
// config/ColdBox.cfc
moduleSettings = {
    qb: {
        defaultGrammar:    "MySQLGrammar",   // MySQLGrammar | PostgresGrammar | MSSQLGrammar | OracleGrammar
        defaultDatasource: "appDB",
        returnFormat:      "query"           // "query" | "array"
    }
}
```

## Injection

```boxlang
property name="qb" inject="QueryBuilder@qb"
```

## SELECT Queries

```boxlang
// All rows
qb.from( "users" ).get()

// Specific columns
qb.select( "id", "firstName", "email" ).from( "users" ).get()

// WHERE
qb.from( "users" ).where( "active", 1 ).get()
qb.from( "users" ).where( "age", ">=", 21 ).get()

// OR conditions
qb.from( "users" )
    .where( "role", "admin" )
    .orWhere( "role", "manager" )
    .get()

// WHERE IN / NOT IN
qb.from( "users" ).whereIn( "role", [ "admin", "manager" ] ).get()
qb.from( "users" ).whereNotIn( "status", [ "deleted", "banned" ] ).get()

// WHERE NULL / NOT NULL
qb.from( "users" ).whereNull( "deletedAt" ).get()
qb.from( "users" ).whereNotNull( "emailVerifiedAt" ).get()

// WHERE BETWEEN
qb.from( "orders" ).whereBetween( "total", [ 100, 500 ] ).get()

// WHERE LIKE
qb.from( "users" ).where( "email", "like", "%@gmail.com" ).get()

// WHERE with closure (grouped conditions)
qb.from( "users" )
    .where( "active", 1 )
    .where( function( q ) {
        q.where( "role", "admin" )
         .orWhere( "role", "manager" )
    } )
    .get()
```

## ORDER BY, LIMIT, PAGINATION

```boxlang
qb.from( "users" ).orderBy( "lastName" ).get()
qb.from( "users" ).orderBy( "createdAt", "desc" ).get()
qb.from( "users" ).limit( 10 ).offset( 20 ).get()

// Helper for page-based pagination
qb.from( "users" ).forPage( page: 3, maxRows: 25 ).get()
```

## GROUP BY / HAVING / DISTINCT

```boxlang
// Distinct
qb.from( "users" ).distinct().select( "department" ).get()

// Group by with aggregate
qb.from( "orders" )
    .select( "userId" )
    .selectRaw( "SUM(total) as totalSpent" )
    .groupBy( "userId" )
    .having( "totalSpent", ">", 1000 )
    .get()
```

## JOINs

```boxlang
// Inner join
qb.from( "users" )
    .join( "orders", "users.id", "orders.userId" )
    .select( "users.*", "orders.total" )
    .get()

// Left join
qb.from( "users" )
    .leftJoin( "orders", "users.id", "orders.userId" )
    .get()

// Join with closure (multi-condition or where)
qb.from( "users" )
    .join( "orders", function( j ) {
        j.on( "users.id", "orders.userId" )
         .where( "orders.status", "completed" )
    } )
    .get()

// Subquery join
qb.from( "users" )
    .joinSub( "recentOrders", function( q ) {
        q.from( "orders" )
         .where( "createdAt", ">=", dateAdd( "d", -30, now() ) )
    }, "users.id", "recentOrders.userId" )
    .get()
```

## Aggregates

```boxlang
var userCount    = qb.from( "users" ).count()
var totalRevenue = qb.from( "orders" ).sum( "total" )
var avgOrder     = qb.from( "orders" ).avg( "total" )
var minPrice     = qb.from( "products" ).min( "price" )
var maxPrice     = qb.from( "products" ).max( "price" )

// Aggregate with filter
var activeCount = qb.from( "users" ).where( "active", 1 ).count()
```

## INSERT / UPDATE / DELETE

```boxlang
// Insert
qb.table( "users" ).insert( {
    firstName: "John",
    lastName:  "Doe",
    email:     "john@example.com",
    createdAt: now()
} )

// Insert multiple
qb.table( "users" ).insert( [
    { firstName: "Alice", email: "alice@example.com" },
    { firstName: "Bob",   email: "bob@example.com" }
] )

// Update
qb.table( "users" )
    .where( "id", 1 )
    .update( { email: "new@example.com", updatedAt: now() } )

// Update or Insert (upsert)
qb.table( "users" )
    .updateOrInsert(
        filter: { email: "john@example.com" },
        values: { firstName: "John", updatedAt: now() }
    )

// Delete
qb.table( "users" ).where( "id", 1 ).delete()

// Truncate
qb.table( "users" ).truncate()
```

## Raw Expressions

```boxlang
// selectRaw
qb.from( "users" )
    .selectRaw( "CONCAT(firstName, ' ', lastName) as fullName" )
    .get()

// whereRaw
qb.from( "users" )
    .whereRaw( "YEAR(createdAt) = ?", [ year( now() ) ] )
    .get()

// orderByRaw
qb.from( "users" )
    .orderByRaw( "FIELD(status, 'active', 'pending', 'inactive')" )
    .get()
```

## Subqueries

```boxlang
// Subquery in WHERE
qb.from( "users" )
    .whereIn( "id", function( q ) {
        q.select( "userId" ).from( "orders" ).where( "total", ">", 500 )
    } )
    .get()

// Subquery as column
qb.from( "users" )
    .addSubSelect( "orderCount", function( q ) {
        q.from( "orders" ).whereColumn( "userId", "users.id" ).count()
    } )
    .get()
```

## Scoped Queries (Reusable)

```boxlang
// Service method using QB
class singleton {
    property name="qb" inject="QueryBuilder@qb"

    function activeUsers() {
        return qb.from( "users" )
            .where( "active", 1 )
            .whereNull( "deletedAt" )
    }

    function getAdmins() {
        return activeUsers()
            .where( "role", "admin" )
            .orderBy( "name" )
            .get()
    }
}
```

## QB Method Quick Reference

| Method | Purpose |
|--------|---------|
| `from( table )` | Set table name |
| `select( ...cols )` | Column selection |
| `selectRaw( sql )` | Raw SELECT expression |
| `where( col, op, val )` | AND WHERE condition |
| `orWhere( col, val )` | OR WHERE condition |
| `whereIn( col, list )` | WHERE IN list |
| `whereBetween( col, [ lo, hi ] )` | WHERE BETWEEN |
| `whereNull( col )` | WHERE IS NULL |
| `join( table, col1, col2 )` | INNER JOIN |
| `leftJoin( table, col1, col2 )` | LEFT JOIN |
| `orderBy( col, dir )` | ORDER BY |
| `groupBy( col )` | GROUP BY |
| `having( col, op, val )` | HAVING |
| `limit( n )` | LIMIT rows |
| `offset( n )` | OFFSET rows |
| `forPage( page, maxRows )` | Pagination helper |
| `count( col )` | COUNT aggregate |
| `sum( col )` | SUM aggregate |
| `avg( col )` | AVG aggregate |
| `get()` | Execute and return all |
| `first()` | Execute and return first |
| `find( id )` | WHERE id = :id, first |
| `insert( data )` | INSERT row(s) |
| `update( data )` | UPDATE matching rows |
| `delete()` | DELETE matching rows |
