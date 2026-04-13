---
name: quick
description: >
  Use this skill when building Active Record-style ORM models in ColdBox/BoxLang with Quick. Covers
  entity definition, CRUD operations, relationships (hasOne, hasMany, belongsTo, belongsToMany),
  query scopes, eager loading, accessors/mutators, global scopes, lifecycle hooks, and production
  patterns for service-layer data access.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# Quick Skill

## When to Use This Skill

Load this skill when:
- Building data models using the Active Record pattern
- Defining and traversing entity relationships (hasOne, hasMany, belongsTo, belongsToMany)
- Writing reusable query scopes on entities
- Eager-loading related data to prevent N+1 queries
- Using lifecycle hooks (before/after create/save/delete)

## Installation

```bash
box install quick
```

## Quick requires qb — install both:

```bash
box install quick,qb
```

## Entity Definition

```js
// models/entities/User.bx
class extends="quick.models.BaseEntity" table="users" {

    // Explicitly list fillable columns (prevents mass-assignment vulnerability)
    variables._fillable = [ "firstName", "lastName", "email", "role" ];

    // Columns never returned in serialization (even if requested)
    variables._hidden = [ "password", "rememberToken" ];

    // Cast column types automatically
    variables._casts = {
        isActive    : "boolean",
        preferences : "json",
        createdAt   : "datetime"
    };

    // ---- Relationships ----

    // User HAS MANY Posts
    function posts() {
        return hasMany( "Post" )
    }

    // User HAS ONE Profile
    function profile() {
        return hasOne( "Profile" )
    }

    // User BELONGS TO Role entity
    function roleEntity() {
        return belongsTo( "Role", "role" )
    }

    // User BELONGS TO MANY Permissions (pivot)
    function permissions() {
        return belongsToMany( "Permission", "user_permissions" )
    }

    // ---- Scopes ----

    function scopeActive( qb ) {
        qb.where( "isActive", true )
    }

    function scopeAdmins( qb ) {
        qb.where( "role", "admin" )
    }

    function scopeSearch( qb, term ) {
        qb.where( function( q ) {
            q.where( "firstName", "like", "%#term#%" )
             .orWhere( "lastName", "like", "%#term#%" )
             .orWhere( "email",    "like", "%#term#%" )
        } )
    }

    // ---- Accessors / Mutators ----

    // Computed property: fullName
    function getFullNameAttribute() {
        return trim( getFirstName() & " " & getLastName() )
    }

    // Hash password on set
    function setPasswordAttribute( value ) {
        return bcrypt.hashPassword( value )
    }
}
```

## CRUD Operations

```js
// Inject entity (WireBox ID = entity class name)
property name="userEntity" inject="User@quick";

// Create
var user = userEntity.create( {
    firstName : "Alice",
    lastName  : "Smith",
    email     : "alice@example.com",
    password  : "secret"   // triggers setPasswordAttribute mutator
} )

// Find by PK
var user = userEntity.find( rc.id )          // returns null if not found
var user = userEntity.findOrFail( rc.id )    // throws EntityNotFound

// Find by attribute
var user = userEntity.where( "email", rc.email ).first()

// Update
user.update( { firstName: "Alice", lastName: "Johnson" } )

// Fill + save
user.fill( rc )
user.save()

// Delete
user.delete()

// Soft delete (requires deletedAt column)
user.softDelete()
user.restore()
```

## Querying

```js
// All active users, newest first
var users = userEntity
    .active()                   // scope
    .orderByDesc( "createdAt" )
    .limit( 25 )
    .get()

// Search
var users = userEntity.search( rc.term ).get()

// Admins
var admins = userEntity.admins().get()

// Count
var total = userEntity.active().count()

// Paginate
var users = userEntity.active().paginate( rc.page ?: 1, 25 )
// returns { data: [], total: n, perPage: 25, currentPage: n, lastPage: n }
```

## Relationships

```js
// Access related entity
var posts    = user.posts().get()          // hasMany
var profile  = user.profile().first()     // hasOne
var perms    = user.permissions().get()   // belongsToMany

// Create through relationship
var post = user.posts().create( { title: "Hello World", body: rc.body } )

// Attach/detach pivot
user.permissions().attach( permissionId )
user.permissions().detach( permissionId )
user.permissions().sync( permissionIds )
```

## Eager Loading (Prevent N+1)

```js
// Without eager loading — N+1 queries for posts
var users = userEntity.all()
for ( u in users ) {
    writeDump( u.posts().get() )   // query per user!
}

// With eager loading — 2 queries total
var users = userEntity.with( "posts" ).get()
var users = userEntity.with( [ "posts", "profile", "permissions" ] ).get()

// Nested eager loading
var users = userEntity.with( "posts.comments" ).get()
```

## Lifecycle Hooks

```js
class extends="quick.models.BaseEntity" {

    function beforeCreate( entity ) {
        entity.setCreatedAt( now() )
    }

    function afterCreate( entity ) {
        welcomeEmailJob.setUserId( entity.getId() ).dispatch()
    }

    function beforeDelete( entity ) {
        // Log the deletion for audit trail
        auditService.logDelete( "User", entity.getId() )
    }
}
```

## Best Practices

- **Always define `_fillable`** — prevents mass-assignment vulnerabilities from untrusted input
- **Always define `_hidden`** — ensures passwords and tokens are never serialized
- **Use scopes for common query patterns** — keeps handler code clean
- **Eager load relationships** when iterating results — prevents N+1 query problems
- **Use `findOrFail()`** instead of `find()` when the record must exist — throws typed exception
- **Use lifecycle hooks** for consistent side effects (audit logs, event dispatch) rather than duplicating in handlers
- **Keep entities as data layer only** — avoid calling handlers or event listeners from inside entities

## Documentation

- quick: https://github.com/coldbox-modules/quick
- quick docs: https://quick.ortusbooks.com
