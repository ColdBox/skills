---
name: cborm
description: >
  Use this skill when working with Hibernate ORM in ColdBox/BoxLang using the cborm module. Covers
  BaseORMService, VirtualEntityService, the Criteria Builder (restrictions, projections, joins,
  sub-queries, pagination), transaction management, event interceptions, and production patterns
  for data access layers.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# CBORM Skill

## When to Use This Skill

Load this skill when:
- Building data-access code backed by Hibernate ORM entities
- Using `BaseORMService` or `VirtualEntityService` for CRUD operations
- Constructing complex queries with the Criteria Builder
- Managing ORM transactions, session flushing, or pagination
- Writing ORM event interceptors

## Installation

```bash
box install cborm
```

## Core API

### BaseORMService (extend in your service)

```js
// services/UserService.bx
class extends="cborm.models.BaseORMService" {

    UserService() {
        super.init( entityName = "User" )
    }
}
```

### VirtualEntityService (no extension needed)

```js
// Inject without extending
property name="userService" inject="entityService:User";
```

### Common CRUD Methods

```js
// Get by PK
var user = userService.get( 1 )

// Get or fail (throws EntityNotFound)
var user = userService.getOrFail( 1 )

// List all
var users = userService.list()

// Count
var total = userService.count()

// Save / merge
userService.save( userEntity )

// Delete
userService.delete( userEntity )

// Delete by criteria
userService.deleteWhere( isActive = false )

// Find by property
var user = userService.findWhere( { email: "john@example.com" } )

// List by criteria
var users = userService.findAllWhere(
    criteria  = { role: "admin" },
    sortOrder = "lastName ASC"
)
```

### Pagination

```js
var result = userService.list(
    criteria    = { isActive: true },
    sortOrder   = "createdAt DESC",
    offset      = rc.offset ?: 0,
    max         = rc.max    ?: 25,
    asQuery     = false
)
// result.count, result.records
```

## Criteria Builder

```js
var c = userService.newCriteria()

var users = c
    .isTrue( "isActive" )
    .like( "email", "%@example.com%" )
    .between( "age", 18, 65 )
    .or(
        c.restrictions.eq( "role", "admin" ),
        c.restrictions.eq( "role", "manager" )
    )
    .order( "lastName", "asc" )
    .list( max = 25, offset = 0 )
```

### Restrictions Reference

| Method | Example |
|--------|---------|
| `eq( prop, val )` | Equals |
| `ne( prop, val )` | Not equals |
| `gt( prop, val )` | Greater than |
| `lt( prop, val )` | Less than |
| `between( prop, lo, hi )` | Range |
| `like( prop, pattern )` | SQL LIKE |
| `isNull( prop )` | IS NULL |
| `isNotNull( prop )` | IS NOT NULL |
| `isTrue( prop )` | Boolean true |
| `isFalse( prop )` | Boolean false |
| `in( prop, array )` | IN list |
| `conjunction( array )` | AND group |
| `disjunction( array )` | OR group |

### Projections

```js
// Count query
var total = userService.newCriteria()
    .isTrue( "isActive" )
    .withProjections( count = "id" )
    .get()

// Select specific columns
var emails = userService.newCriteria()
    .isNotNull( "email" )
    .withProjections( property = "email,firstName" )
    .list()
```

### Associations

```js
// Inner join to access related entity properties
var orders = orderService.newCriteria()
    .createAlias( "user", "u" )
    .eq( "u.role", "customer" )
    .gt( "total", 100 )
    .list()
```

## Production Patterns

### Transactional Service Method

```js
@transactional
function transferFunds( fromId, toId, amount ) {
    var from = getOrFail( fromId )
    var to   = getOrFail( toId )

    if ( from.balance < amount ) {
        throw( type = "InsufficientFunds", message = "Insufficient balance." )
    }

    from.setBalance( from.balance - amount )
    to.setBalance(   to.balance   + amount )

    save( from )
    save( to )
}
```

### Repository Pattern

```js
class extends="cborm.models.BaseORMService" {

    PostService() {
        super.init( entityName = "Post" )
    }

    // Published posts, newest first
    array function getPublished( max = 10, offset = 0 ) {
        return newCriteria()
            .eq( "status", "published" )
            .isNotNull( "publishedAt" )
            .order( "publishedAt", "desc" )
            .list( max = max, offset = offset )
    }

    // Search by title/body
    array function search( term, max = 20 ) {
        return newCriteria()
            .or(
                newCriteria().restrictions.like( "title", "%#term#%" ),
                newCriteria().restrictions.like( "body",  "%#term#%" )
            )
            .eq( "status", "published" )
            .list( max = max )
    }
}
```

## Legacy ORM Coverage (Merged from former `orm/cborm`)

### Language Mode Reference

| Concept | BoxLang (`.bx`) | CFML (`.cfc`) |
|---|---|---|
| Class declaration | `class [extends="..."] {` | `component [extends="..."] {` |
| DI annotation | `@inject` + `property` | `property ... inject="...";` |
| View templates | `.bxm` | `.cfm` / `.cfml` |

### Application ORM Setup

```boxlang
this.ormEnabled  = true
this.ormSettings = {
    cfclocation:       [ "/models" ],
    dbcreate:          "update",
    flushAtRequestEnd: false,
    autoManageSession: false,
    eventHandling:     true,
    eventHandler:      "cborm.models.EventHandler"
}
```

### ActiveEntity Pattern

```boxlang
class extends="cborm.models.ActiveEntity" {
    property name="id" fieldtype="id" generator="uuid"
    property name="email" unique="true"

    function findActive() {
        return newCriteria().eq( "active", true ).list( sortOrder: "lastName" )
    }
}
```

### ORM Event Interceptor Example

```boxlang
class {
    function preInsert( entity ) {
        entity.setCreatedAt( now() )
    }

    function preUpdate( entity ) {
        entity.setUpdatedAt( now() )
    }
}
```

## Best Practices

- **Prefer `getOrFail()`** over `get()` when the record must exist — throws a typed exception you can catch
- **Use `@transactional` on service methods** that span multiple entity saves
- **Always paginate** large queries — never load unbounded result sets
- **Use `asQuery = false`** in `list()` to get typed entity arrays rather than query objects
- **Avoid N+1 queries** — use `createAlias` joins or Hibernate `fetch = "join"` on associations
- **Flush ORM session sparingly** — rely on Hibernate's unit-of-work rather than calling `ORMFlush()` manually
- **Keep entities thin** — business logic belongs in service layer, not in entity methods

## Documentation

- cborm: https://github.com/coldbox-modules/cborm
- ColdBox ORM docs: https://coldbox.ortusbooks.com/digging-deeper/orm-integration
