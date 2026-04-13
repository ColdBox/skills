---
name: qb
description: >
  Use this skill when building database queries with qb (Query Builder) in ColdBox/BoxLang. Covers
  QueryBuilder injection, select/from/where/join/group/order/limit clauses, aggregates, inserts,
  updates, deletes, raw expressions, sub-queries, chunking, and grammar configuration for MySQL,
  Postgres, MSSQL, and SQLite.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# QB Skill

## When to Use This Skill

Load this skill when:
- Writing SQL queries using a fluent, chainable builder (no raw SQL)
- Performing JOINs, sub-queries, or complex WHERE conditions
- Inserting, updating, or deleting records safely with parameter binding
- Paginating large result sets
- Using database-agnostic code that works across MySQL, Postgres, MSSQL, and SQLite

## Installation

```bash
box install qb
```

## Configuration

### config/modules/qb.cfc

```js
function configure() {
    return {
        // Default grammar matching your database engine
        defaultGrammar : "MySQLGrammar@qb",   // MySQLGrammar | PostgresGrammar | MSSQLGrammar | SQLiteGrammar

        // Datasource to use
        defaultDatasource : "myDatasource",

        // Return queries as arrays of structs (recommended)
        returnFormat : "array"
    }
}
```

## Core API

### Injection

```js
property name="qb" inject="QueryBuilder@qb";
```

### SELECT Queries

```js
// All records
var users = qb.from( "users" ).get()

// Specific columns
var users = qb.select( "id,name,email" ).from( "users" ).get()

// With alias
var users = qb.select( [ "id", "email", { "full_name": "name" } ] ).from( "users" ).get()

// Distinct
var roles = qb.distinct().select( "role" ).from( "users" ).get()

// First record
var user = qb.from( "users" ).where( "id", rc.id ).first()

// Get the value of a single column
var email = qb.from( "users" ).where( "id", 1 ).value( "email" )
```

### WHERE Clauses

```js
qb.from( "users" )
    .where( "isActive", true )
    .where( "role", "=", "admin" )
    .whereIn( "id", [ 1, 2, 3 ] )
    .whereNotIn( "status", [ "banned", "deleted" ] )
    .whereNull( "deletedAt" )
    .whereNotNull( "emailVerifiedAt" )
    .whereBetween( "age", 18, 65 )
    .where( "email", "like", "%@example.com%" )
    .get()

// OR conditions
qb.from( "users" )
    .where( "role", "admin" )
    .orWhere( "role", "superadmin" )
    .get()

// Grouped WHERE
qb.from( "posts" )
    .where( function( q ) {
        q.where( "status", "published" )
         .orWhere( "featured", true )
    } )
    .where( "userId", rc.userId )
    .get()
```

### ORDER, LIMIT, OFFSET

```js
qb.from( "users" )
    .orderBy( "lastName",  "asc" )
    .orderBy( "firstName", "asc" )
    .limit( 25 )
    .offset( 50 )
    .get()

// Shorthand
qb.from( "users" ).orderByDesc( "createdAt" ).limit( 10 ).get()
```

### Aggregates

```js
qb.from( "orders" ).where( "status", "completed" ).count()
qb.from( "orders" ).sum( "total" )
qb.from( "products" ).avg( "price" )
qb.from( "products" ).max( "price" )
qb.from( "products" ).min( "price" )
```

### JOINs

```js
// INNER JOIN
qb.from( "posts" )
    .join( "users", "users.id", "=", "posts.userId" )
    .select( "posts.id,posts.title,users.name AS authorName" )
    .get()

// LEFT JOIN
qb.from( "users" )
    .leftJoin( "profiles", "profiles.userId", "=", "users.id" )
    .whereNull( "profiles.id" )   // users without profiles
    .get()

// Complex join
qb.from( "orders" )
    .join( "order_items", function( join ) {
        join.on( "order_items.orderId", "=", "orders.id" )
            .where( "order_items.qty", ">", 0 )
    } )
    .get()
```

### GROUP BY / HAVING

```js
qb.from( "orders" )
    .select( "userId" )
    .selectRaw( "COUNT(*) AS orderCount, SUM(total) AS totalSpent" )
    .groupBy( "userId" )
    .having( "orderCount", ">", 5 )
    .orderByDesc( "totalSpent" )
    .get()
```

### INSERT

```js
// Single
qb.table( "users" ).insert( {
    name     : rc.name,
    email    : rc.email,
    password : hashedPassword,
    role     : "user"
} )

// Multiple rows
qb.table( "users" ).insert( [
    { name: "Alice", email: "alice@example.com" },
    { name: "Bob",   email: "bob@example.com" }
] )
```

### UPDATE

```js
qb.table( "users" )
    .where( "id", rc.id )
    .update( {
        name  : rc.name,
        email : rc.email
    } )
```

### DELETE

```js
qb.table( "users" ).where( "id", rc.id ).delete()
qb.table( "sessions" ).where( "expiredAt", "<", now() ).delete()
```

### Sub-queries

```js
// WHERE IN subquery
qb.from( "posts" )
    .whereIn( "userId", function( q ) {
        q.select( "id" ).from( "users" ).where( "role", "admin" )
    } )
    .get()
```

### Raw Expressions

```js
qb.from( "products" )
    .selectRaw( "id, name, price * 1.1 AS priceWithTax" )
    .whereRaw( "MATCH(name, description) AGAINST (? IN BOOLEAN MODE)", [ rc.term ] )
    .get()
```

### Pagination (with cbpaginator)

```js
var total = qb.from( "users" ).where( "isActive", true ).count()

var page  = paginator.paginate( page = rc.page ?: 1, maxRows = 25, totalRecords = total )

var users = qb.from( "users" )
    .where( "isActive", true )
    .orderByDesc( "createdAt" )
    .limit( page.getMaxRows() )
    .offset( page.getOffset() )
    .get()
```

### Chunk (large datasets)

```js
qb.from( "users" )
    .where( "subscribed", true )
    .chunk( 100, function( rows ) {
        for ( var row in rows ) {
            emailService.sendNewsletter( row.email )
        }
    } )
```

## Best Practices

- **Use `where()` parameter binding** — qb always parameterizes values, preventing SQL injection
- **Prefer column allowlists** in `select()` over `select("*")` — avoid inadvertently exposing new columns
- **Use `first()` for single-row lookups** — throws an exception if no row found, avoids null checks
- **Paginate all list queries** — never load unbounded result sets
- **Use `chunk()`** for batch processing of large tables — avoids OOM errors
- **Avoid `whereRaw`** unless absolutely necessary — prefer typed `where*()`methods to maintain safety
- **Configure the correct grammar** for your database — different grammars generate different SQL dialects

## Documentation

- qb: https://github.com/coldbox-modules/qb
- qb docs: https://qb.ortusbooks.com
