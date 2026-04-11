---
name: coldbox-orm-quick
description: "Use this skill when using Quick ORM in ColdBox, defining Active Record entities, setting up entity relationships (hasMany/belongsTo/belongsToMany/hasOne), querying with the Quick fluent API, using eager loading with 'with()', defining query scopes, or using entity accessors and mutators."
---

# Quick ORM in ColdBox

## Overview

Quick is a modern, fluent Active Record ORM for CFML/BoxLang. It provides an expressive syntax for database operations, relationships, and query building — similar to Laravel Eloquent.

## Language Mode Reference

Examples use **BoxLang (`.bx`)** syntax by default. Adapt for your target language:

| Concept | BoxLang (`.bx`) | CFML (`.cfc`) |
|---------|-----------------|---------------|
| Class declaration | `class [extends="..."] {` | `component [extends="..."] {` |
| DI annotation | `@inject` above `property name="svc";` | `property name="svc" inject="svc";` |
| View templates | `.bxm` suffix | `.cfm` / `.cfml` suffix |
| Tag prefix | `<bx:if>`, `<bx:output>`, `<bx:set>` | `<cfif>`, `<cfoutput>`, `<cfset>` |

> **CFML Compat Mode**: With BoxLang + CFML Compat module, `.bx` and `.cfc` files coexist freely. BoxLang-native classes use `class {}` (`.bx` files); CFML-compat classes use `component {}` (`.cfc` files).

## Installation & Configuration

```bash
box install quick
```

```boxlang
// config/ColdBox.cfc
moduleSettings = {
    quick: {
        defaultGrammar:    "MySQLGrammar@qb",
        defaultDatasource: "appDB"
    }
}
```

## Entity Definition

```boxlang
/**
 * models/User.cfc
 */
class extends="quick.models.BaseEntity" {

    variables.table     = "users"     // optional — defaults to pluralized class name
    variables.key       = "id"        // optional — defaults to "id"

    variables.fillable  = [ "firstName", "lastName", "email", "password" ]
    variables.hidden    = [ "password" ]  // excluded from serialization

    variables.casts     = {
        isActive:  "boolean",
        createdAt: "datetime",
        settings:  "json"
    }
}
```

## Relationships

```boxlang
class extends="quick.models.BaseEntity" {

    variables.fillable = [ "name", "email" ]

    // One-to-Many
    function posts() {
        return hasMany( "Post" )
    }

    // Many-to-One
    function role() {
        return belongsTo( "Role" )
    }

    // One-to-One
    function profile() {
        return hasOne( "Profile" )
    }

    // Many-to-Many (pivot table)
    function teams() {
        return belongsToMany( "Team" )
    }

    // Has-many-through
    function countryUsers() {
        return hasManyThrough( [ "Country", "State" ] )
    }
}
```

## CRUD Operations

### Create

```boxlang
// Fluent create
var user = getInstance( "User" ).create( {
    firstName: "John",
    lastName:  "Doe",
    email:     "john@example.com"
} )

// Setter-based
var user = getInstance( "User" )
user.setFirstName( "John" )
user.setEmail( "john@example.com" )
user.save()

// forceFill ignores the fillable list
var user = getInstance( "User" ).forceFill( {
    id:       createUUID(),
    password: hashedPassword
} ).save()
```

### Read

```boxlang
// By primary key
var user = getInstance( "User" ).find( 1 )

// Or throws EntityNotFound
var user = getInstance( "User" ).findOrFail( 1 )

// First matching condition
var user = getInstance( "User" )
    .where( "email", "john@example.com" )
    .first()

// All records
var users = getInstance( "User" ).all()

// With constraints
var active = getInstance( "User" )
    .where( "isActive", true )
    .orderBy( "createdAt", "desc" )
    .get()

// Pagination
var page = getInstance( "User" )
    .orderBy( "name" )
    .paginate( page: 1, maxRows: 25 )
// page.results, page.totalCount, page.pages
```

### Update

```boxlang
var user = getInstance( "User" ).findOrFail( 1 )
user.setEmail( "new@example.com" )
user.save()

// Update with struct
user.update( { email: "new@example.com" } )

// Bulk update
getInstance( "User" )
    .where( "role", "trial" )
    .update( { isActive: false } )
```

### Delete

```boxlang
var user = getInstance( "User" ).findOrFail( 1 )
user.delete()

// Bulk delete
getInstance( "User" )
    .where( "isActive", false )
    .delete()
```

## Querying

```boxlang
// Basic where
getInstance( "User" ).where( "role", "admin" ).get()

// Where with operator
getInstance( "User" ).where( "age", ">=", 18 ).get()

// OR conditions
getInstance( "User" )
    .where( "role", "admin" )
    .orWhere( "role", "manager" )
    .get()

// WHERE IN
getInstance( "User" )
    .whereIn( "role", [ "admin", "manager" ] )
    .get()

// NULL checks
getInstance( "User" ).whereNull( "deletedAt" ).get()
getInstance( "User" ).whereNotNull( "emailVerifiedAt" ).get()

// ORDER BY and LIMIT
getInstance( "User" ).orderBy( "lastName" ).limit( 10 ).get()

// Count
var total = getInstance( "User" ).where( "active", 1 ).count()

// First or create
var user = getInstance( "User" ).firstOrCreate(
    search: { email: "john@example.com" },
    create: { firstName: "John", email: "john@example.com" }
)
```

## Eager Loading (Prevent N+1)

```boxlang
// Load users with their role (1 extra query instead of N)
var users = getInstance( "User" )
    .with( "role" )
    .get()

// Load multiple relations
var users = getInstance( "User" )
    .with( [ "role", "profile", "posts" ] )
    .get()

// Nested eager loading
var users = getInstance( "User" )
    .with( "posts.comments" )
    .get()

// Constrained eager loading
var users = getInstance( "User" )
    .with( {
        posts: ( q ) => q.where( "published", true ).orderBy( "createdAt", "desc" )
    } )
    .get()
```

## Query Scopes

```boxlang
/**
 * Define reusable query scopes on an entity
 */
class extends="quick.models.BaseEntity" {

    // Scope: scopeActive( query )
    function scopeActive( q ) {
        return q.where( "isActive", true ).whereNull( "deletedAt" )
    }

    // Scope: scopeAdmins( query )
    function scopeAdmins( q ) {
        return q.where( "role", "admin" )
    }

    // Scope with parameter: scopeRole( query, role )
    function scopeRole( q, required role ) {
        return q.where( "role", arguments.role )
    }
}

// Usage
getInstance( "User" ).active().get()
getInstance( "User" ).admins().get()
getInstance( "User" ).role( "manager" ).get()

// Combine scopes
getInstance( "User" ).active().role( "admin" ).orderBy( "name" ).get()
```

## Accessors & Mutators

```boxlang
class extends="quick.models.BaseEntity" {

    // Accessor — transform when reading
    function getFullNameAttribute() {
        return variables.firstName & " " & variables.lastName
    }

    // Mutator — transform before storage
    function setPasswordAttribute( required value ) {
        variables.password = bcrypt.hashPassword( arguments.value )
    }
}

// Usage
var user = getInstance( "User" ).findOrFail( 1 )
writeOutput( user.getFullName() )  // accessor
user.setPassword( "secret123" )    // mutator — auto-hashed
user.save()
```

## Relationships in Practice

```boxlang
// Access related entity
var user = getInstance( "User" ).with( "role" ).findOrFail( 1 )
var roleName = user.role().name

// Create related entity
var user = getInstance( "User" ).findOrFail( 1 )
user.posts().create( { title: "Hello World", content: "..." } )

// Many-to-many — attach/detach
var user = getInstance( "User" ).findOrFail( 1 )
user.teams().attach( teamID )
user.teams().detach( teamID )
user.teams().sync( [ teamID1, teamID2 ] )  // replace all associations
```

## Quick Method Quick Reference

| Method | Purpose |
|--------|---------|
| `find( id )` | Load by PK or null |
| `findOrFail( id )` | Load by PK or throw |
| `first()` | First matching row |
| `firstOrCreate( search, create )` | Find or create |
| `all()` | All rows |
| `get()` | Execute query, return array |
| `create( struct )` | Insert and return entity |
| `update( struct )` | Update and return entity |
| `delete()` | Delete entity or bulk |
| `save()` | Persist changes |
| `where( col, op, val )` | Add WHERE condition |
| `orWhere( col, val )` | OR WHERE condition |
| `whereIn( col, list )` | WHERE IN |
| `whereNull( col )` | IS NULL |
| `with( relations )` | Eager load relations |
| `orderBy( col, dir )` | ORDER BY |
| `limit( n )` | LIMIT |
| `paginate( page, max )` | Paginated results |
| `count()` | COUNT aggregate |
| `hasMany( entity )` | One-to-many relationship |
| `belongsTo( entity )` | Many-to-one relationship |
| `hasOne( entity )` | One-to-one relationship |
| `belongsToMany( entity )` | Many-to-many relationship |
