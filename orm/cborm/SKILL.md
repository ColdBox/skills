---
name: coldbox-orm-cborm
description: "Use this skill when using cborm ORM utilities in ColdBox with Hibernate, extending BaseORMService or ActiveEntity, using criteria queries and DetachedCriteria, building ORM-powered virtual entity services, handling ORM events, or performing HQL queries with the cborm module."
---

# CBORM - ORM Utilities in ColdBox

## Overview

cborm is a ColdBox module that provides utility services, the Active Entity pattern, virtual entity services, and Hibernate ORM event handling. It extends native CFML/BoxLang ORM with a clean, consistent API.

## Language Mode Reference

Examples use **BoxLang (`.bx`)** syntax by default. Adapt for your target language:

| Concept | BoxLang (`.bx`) | CFML (`.cfc`) |
|---------|-----------------|---------------|
| Class declaration | `class [extends="..."] {` | `component [extends="..."] {` |
| DI annotation | `@inject` above `property name="svc";` | `property name="svc" inject="svc";` |
| View templates | `.bxm` suffix | `.cfm` / `.cfml` suffix |
| Tag prefix | `<bx:if>`, `<bx:output>`, `<bx:set>` | `<cfif>`, `<cfoutput>`, `<cfset>` |

> **CFML Compat Mode**: With BoxLang + CFML Compat module, `.bx` and `.cfc` files coexist freely. BoxLang-native classes use `class {}` (`.bx` files); CFML-compat classes use `component {}` (`.cfc` files).

## Installation & Setup

```bash
box install cborm
```

### Application.cfc

```boxlang
this.ormEnabled  = true
this.ormSettings = {
    cfclocation:          [ "/models" ],
    dbcreate:             "update",
    logSQL:               true,
    flushAtRequestEnd:    false,
    autoManageSession:    false,
    eventHandling:        true,
    eventHandler:         "cborm.models.EventHandler"
}
```

### config/ColdBox.cfc

```boxlang
moduleSettings = {
    cborm: {
        injection: {
            enabled: true
        }
    }
}
```

## BaseORMService

```boxlang
/**
 * models/UserService.cfc
 */
class extends="cborm.models.BaseORMService" singleton {

    function init() {
        setEntityName( "User" )
        return this
    }

    function getActiveUsers() {
        return newCriteria()
            .eq( "active", true )
            .list( sortOrder: "lastName" )
    }

    function findByEmail( required email ) {
        return newCriteria().eq( "email", arguments.email ).get()
    }
}
```

### Common BaseORMService Methods

```boxlang
// CRUD
var entity = super.save( entity )
super.delete( entity )
super.deleteByID( id )

// Retrieval
var entity = super.get( id )           // find by PK (returns null if not found)
var entity = super.load( id )          // find by PK (throws if not found)
var all    = super.getAll()            // all entities
var all    = super.getAll( [ 1, 2 ] )  // by array of PKs

// Listing
var list = super.list( sortOrder: "name", offset: 0, max: 25 )

// Counts
var total = super.count()
var total = super.countWhere( { active: true } )

// Find by criteria
var entity = super.findWhere( { email: "john@example.com" } )
var list   = super.findAllWhere( { role: "admin" }, "name" )

// HQL
var list = super.executeQuery( "from User where active = :active", { active: true } )
```

## Criteria Query API

```boxlang
// Equality
newCriteria()
    .eq( "status", "active" )
    .list()

// Comparison operators
newCriteria()
    .gt( "age", 18 )       // greater than
    .lt( "age", 65 )       // less than
    .ge( "salary", 50000 ) // greater or equal
    .le( "salary", 100000 )// less or equal
    .list()

// LIKE / NOT LIKE
newCriteria()
    .like( "email", "%@gmail.com" )
    .list()

// IN list
newCriteria()
    .isIn( "role", [ "admin", "manager" ] )
    .list()

// IS NULL / IS NOT NULL
newCriteria()
    .isNull( "deletedAt" )
    .isNotNull( "emailVerifiedAt" )
    .list()

// BETWEEN
newCriteria()
    .between( "age", 18, 65 )
    .list()

// AND (default) and OR groupings
newCriteria()
    .eq( "active", true )
    .disjunction()                  // OR group
        .eq( "role", "admin" )
        .eq( "role", "manager" )
    .end()
    .list()

// Ascending / descending order
newCriteria()
    .eq( "active", true )
    .order( "lastName" )
    .order( "firstName" )
    .list()

// Limit and offset
newCriteria()
    .list( max: 25, offset: 0, sortOrder: "name" )

// Count
var total = newCriteria().eq( "active", true ).count()

// Get single result
var user = newCriteria().eq( "email", "john@example.com" ).get()

// Projections (select specific columns)
newCriteria()
    .withProjections( property: "firstName,lastName,email" )
    .list()

// Association traversal (traverse relations)
newCriteria()
    .createAlias( "role", "r" )
    .eq( "r.name", "admin" )
    .list()
```

## Active Entity Pattern

```boxlang
/**
 * models/User.cfc — entity + service combined
 */
class extends="cborm.models.ActiveEntity" {

    property name="id"        column="id"        fieldtype="id" generator="uuid"
    property name="firstName" column="first_name"
    property name="lastName"  column="last_name"
    property name="email"     column="email"      unique="true"
    property name="active"    column="is_active"  ormtype="boolean"
    property name="createdAt" column="created_at" ormtype="timestamp"

    /**
     * Find active users — instance method on active entity
     */
    function findActive() {
        return newCriteria()
            .eq( "active", true )
            .list( sortOrder: "lastName" )
    }

    /**
     * Custom validation
     */
    function validate() {
        return validateOrFail()
    }
}

// Usage in handler
var user = getInstance( "User" )
user.setFirstName( "John" )
user.setEmail( "john@example.com" )
user.save()

var users = getInstance( "User" ).findActive()
```
**CFML (`.cfc`):**

```cfml
/**
 * models/User.cfc — entity + service combined
 */
component extends="cborm.models.ActiveEntity" {

    property name="id"        column="id"        fieldtype="id" generator="uuid"
    property name="firstName" column="first_name"
    property name="lastName"  column="last_name"
    property name="email"     column="email"      unique="true"
    property name="active"    column="is_active"  ormtype="boolean"
    property name="createdAt" column="created_at" ormtype="timestamp"

    /**
     * Find active users — instance method on active entity
     */
    function findActive() {
        return newCriteria()
            .eq( "active", true )
            .list( sortOrder: "lastName" )
    }

    /**
     * Custom validation
     */
    function validate() {
        return validateOrFail()
    }
}

// Usage in handler
var user = getInstance( "User" )
user.setFirstName( "John" )
user.setEmail( "john@example.com" )
user.save()

var users = getInstance( "User" ).findActive()
```

## Virtual Entity Service

```boxlang
// Inject and use without creating a dedicated service class
class {

    property name="ormService" inject="VirtualEntityService@cborm"

    function getUsers() {
        return ormService
            .newCriteria( "User" )
            .eq( "active", true )
            .list()
    }

    function saveUser( user ) {
        return ormService.save( "User", user )
    }

    function deleteUser( id ) {
        ormService.deleteByID( "User", id )
    }

    function countUsers() {
        return ormService.count( "User" )
    }
}
```
**CFML (`.cfc`):**

```cfml
// Inject and use without creating a dedicated service component component {

    property name="ormService" inject="VirtualEntityService@cborm"

    function getUsers() {
        return ormService
            .newCriteria( "User" )
            .eq( "active", true )
            .list()
    }

    function saveUser( user ) {
        return ormService.save( "User", user )
    }

    function deleteUser( id ) {
        ormService.deleteByID( "User", id )
    }

    function countUsers() {
        return ormService.count( "User" )
    }
}
```

## ORM Events

```boxlang
/**
 * models/UserEventHandler.cfc — listen to ORM lifecycle events
 */
class {

    function preInsert( entity ) {
        entity.setCreatedAt( now() )
    }

    function preUpdate( entity ) {
        entity.setUpdatedAt( now() )
    }

    function preDelete( entity ) {
        // Prevent deletion if user has orders
        var orderCount = queryExecute(
            "SELECT COUNT(*) as cnt FROM orders WHERE user_id = :id",
            { id: entity.getID() }
        ).cnt
        if ( orderCount > 0 ) {
            throw( "Cannot delete user with existing orders" )
        }
    }

    function postInsert( entity ) {
        // Send welcome email
    }
}
```
**CFML (`.cfc`):**

```cfml
/**
 * models/UserEventHandler.cfc — listen to ORM lifecycle events
 */
component {

    function preInsert( entity ) {
        entity.setCreatedAt( now() )
    }

    function preUpdate( entity ) {
        entity.setUpdatedAt( now() )
    }

    function preDelete( entity ) {
        // Prevent deletion if user has orders
        var orderCount = queryExecute(
            "SELECT COUNT(*) as cnt FROM orders WHERE user_id = :id",
            { id: entity.getID() }
        ).cnt
        if ( orderCount > 0 ) {
            throw( "Cannot delete user with existing orders" )
        }
    }

    function postInsert( entity ) {
        // Send welcome email
    }
}
```

## Transaction Management

```boxlang
class singleton {

    // Declare method as transactional
    /**
     * @transaction
     */
    function transferFunds( fromID, toID, amount ) {
        var from = get( fromID )
        var to   = get( toID )

        from.setBalance( from.getBalance() - amount )
        to.setBalance( to.getBalance() + amount )

        save( from )
        save( to )
    }

    // OR use transaction block
    function processOrder( orderID ) {
        transaction {
            try {
                var order = orderService.get( orderID )
                inventoryService.deduct( order )
                order.markComplete()
                orderService.save( order )
                transaction action="commit"
            } catch ( any e ) {
                transaction action="rollback"
                rethrow
            }
        }
    }
}
```
**CFML (`.cfc`):**

```cfml
component singleton {

    // Declare method as transactional
    /**
     * @transaction
     */
    function transferFunds( fromID, toID, amount ) {
        var from = get( fromID )
        var to   = get( toID )

        from.setBalance( from.getBalance() - amount )
        to.setBalance( to.getBalance() + amount )

        save( from )
        save( to )
    }

    // OR use transaction block
    function processOrder( orderID ) {
        transaction {
            try {
                var order = orderService.get( orderID )
                inventoryService.deduct( order )
                order.markComplete()
                orderService.save( order )
                transaction action="commit"
            } catch ( any e ) {
                transaction action="rollback"
                rethrow
            }
        }
    }
}
```

## cborm Criteria Quick Reference

| Method | SQL Equivalent |
|--------|---------------|
| `eq( col, val )` | `col = val` |
| `ne( col, val )` | `col != val` |
| `gt( col, val )` | `col > val` |
| `lt( col, val )` | `col < val` |
| `ge( col, val )` | `col >= val` |
| `le( col, val )` | `col <= val` |
| `like( col, val )` | `col LIKE val` |
| `isIn( col, list )` | `col IN (list)` |
| `isNull( col )` | `col IS NULL` |
| `isNotNull( col )` | `col IS NOT NULL` |
| `between( col, lo, hi )` | `col BETWEEN lo AND hi` |
| `order( col )` | `ORDER BY col ASC` |
| `list( max, offset )` | Paginated list |
| `count()` | `SELECT COUNT(*)` |
| `get()` | `LIMIT 1` |
| `createAlias( rel, alias )` | JOIN traversal |
