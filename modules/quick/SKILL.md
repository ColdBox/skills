---
name: quick
description: >
  Use this skill when building Active Record-style database entities with Quick ORM in ColdBox,
  CFML, or BoxLang applications. Covers entity metadata, safe persistence, relationships, query
  scopes, eager loading, serialization, soft deletes, lifecycle events, and service boundaries.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# Quick ORM

Use this skill when code extends `quick.models.BaseEntity`, injects a `quickService:...`, or needs
to query and persist relational data through Quick. Quick delegates query construction to qb, so
qb methods can be chained from a Quick entity, builder, or relationship unless Quick documents a
different return type.

Check the installed Quick major version before suggesting version-specific APIs. The examples below
use the current Quick API and CFML script syntax, which works on supported CFML engines and the
BoxLang runtime. When editing a native `.bx` entity, preserve that file's BoxLang syntax while using
the same Quick method and metadata contracts.

## Installation and configuration

Install Quick from ForgeBox:

```bash
box install quick
```

Quick declares qb as a package dependency. Install qb separately only when the application uses qb
directly or needs to pin it explicitly.

Configure the grammar in `moduleSettings.quick` when the application does not use the default:

```cfml
moduleSettings = {
    quick = {
        defaultGrammar = "PostgresGrammar@qb"
    }
};
```

Use the grammar that matches the application's database. Quick uses the application's default
datasource unless an entity overrides it with component `datasource="..."` metadata.

## Define an entity

Quick maps declared properties to database columns. It does not use `_fillable`, `_hidden`, or a
struct-valued `_casts` convention.

```cfml
// models/User.cfc
component table="users" extends="quick.models.BaseEntity" accessors="true" {

    property name="bcrypt" inject="@BCrypt" persistent="false";

    property name="id";
    property name="firstName" column="first_name";
    property name="lastName" column="last_name";
    property name="email";
    property name="password";
    property name="roleId" column="role_id";
    property name="isActive" column="is_active" casts="BooleanCast@quick";
    property name="preferences" casts="JsonCast@quick";
    property name="createdDate" column="created_date";
    property name="modifiedDate" column="modified_date";

    // Prevent sensitive data from ever appearing in a memento.
    this.memento = {
        neverInclude = [ "password" ]
    };

    function setPassword( value ) {
        return assignAttribute( "password", bcrypt.hashPassword( arguments.value ) );
    }

    function getFullName() {
        return trim( getFirstName() & " " & getLastName() );
    }
}
```

Important entity metadata:

- Quick assumes table names are pluralized `snake_case` entity names; override with component
  `table="..."` metadata.
- Quick assumes an `id` primary key; override it with `variables._key`, using an array for a
  composite key.
- Declare aliases with `property name="camelCase" column="snake_case"`.
- Use `persistent="false"` for injected or non-database properties.
- Use `readOnly="true"`, `insert="false"`, or `update="false"` to restrict persistence.
- Use `casts="BooleanCast@quick"`, `casts="JsonCast@quick"`, or a custom WireBox cast mapping on
  each property. Do not invent scalar cast names such as `boolean`, `json`, or `datetime`.
- Custom accessors and mutators are named `getAttributeName()` and `setAttributeName()`. Inside
  them, use `retrieveAttribute()` and `assignAttribute()` to avoid recursion.

## Retrieve and persist entities

Start with the entity's application WireBox mapping, usually its component name:

```cfml
property name="userEntity" inject="User";

var user = userEntity.find( rc.id );       // entity or null
var user = userEntity.findOrFail( rc.id ); // throws EntityNotFound

var activeUsers = userEntity
    .where( "isActive", true )
    .orderByDesc( "createdDate" )
    .get();

var createdUser = userEntity.create( {
    firstName = "Alice",
    lastName = "Smith",
    email = "alice@example.com",
    password = rc.password
} );

createdUser.update( { lastName = "Johnson" } );
createdUser.delete();
```

Do not pass an entire request collection to `fill()`, `create()`, or `update()` without filtering
it. Quick's persistent properties are fillable by default. Build an explicit struct or use the
`include` argument:

```cfml
user.fill(
    attributes = rc,
    include = [ "firstName", "lastName", "email" ]
).save();
```

Use `exclude` only when the input keys are otherwise trusted and the denylist is complete.
`ignoreNonExistentAttributes=true` suppresses unknown-attribute errors; it is not a mass-assignment
security boundary.

### Pagination

```cfml
var page = userEntity
    .where( "isActive", true )
    .paginate( page = rc.page ?: 1, maxRows = 25 );

var users = page.results;
var metadata = page.pagination;
```

The result is `{ results, pagination }`. Pagination metadata includes values such as `page`,
`maxRows`, `offset`, `totalRecords`, and `totalPages`; it is not a Laravel-style
`{ data, total, perPage, currentPage, lastPage }` response.

## Query scopes

Define a `scopeName` method whose first argument is the Quick builder. Call it without the `scope`
prefix. Return the builder when useful for readability; returning nothing also keeps the chain.

```cfml
function scopeActive( qb ) {
    return arguments.qb.where( "isActive", true );
}

function scopeSearch( qb, term ) {
    return arguments.qb.where( function( q ) {
        arguments.q
            .where( "firstName", "like", "%#term#%" )
            .orWhere( "lastName", "like", "%#term#%" )
            .orWhere( "email", "like", "%#term#%" );
    } );
}

var users = userEntity.active().search( rc.term ).get();
```

Use attribute aliases in Quick queries. Drop to physical column names only when working directly
with raw qb or SQL, or when an API explicitly requires a column.

For a global scope, override `applyGlobalScopes( qb )` and apply named scopes or qb constraints.
Use `withoutGlobalScope( name )`, or `withoutGlobalScope()` with no name to remove them all, only
for intentional exceptions.

## Relationships

Relationship definitions return Quick relationship objects:

```cfml
function posts() {
    return hasMany( "Post", "user_id" );
}

function profile() {
    return hasOne( "Profile", "user_id" );
}

function role() {
    return belongsTo( "Role", "role_id" );
}

function permissions() {
    return belongsToMany( "Permission", "user_permissions" );
}
```

For a loaded entity, use the generated relationship getter to lazy load and cache the related data,
or call the relationship method when adding query constraints:

```cfml
var posts = user.getPosts();
var recentPosts = user.posts().orderByDesc( "publishedDate" ).get();
var post = user.posts().findOrFail( rc.postId );

var newPost = user.posts().create( {
    title = "Hello World",
    body = rc.body
} );

user.permissions().attach( permissionId );
user.permissions().detach( permissionId );
user.permissions().sync( permissionIds );
```

`hasManyThrough()` accepts an array of relationship method names, not entity names:

```cfml
function permissions() {
    return hasManyThrough( [ "userPermissions", "permission" ] );
}
```

Each name must resolve on the entity produced by the previous relationship.

## Eager loading

Use eager loading before iterating related records:

```cfml
var users = userEntity.with( "posts" ).get();
var users = userEntity.with( [ "posts", "profile", "role" ] ).get();
var users = userEntity.with( "posts.comments" ).get();
```

After eager loading, read relationships with generated getters such as `user.getPosts()`. Calling
`user.posts().get()` creates and executes a relationship query; doing that in a loop recreates the
N+1 query problem.

## Serialization

Quick bundles Mementifier. Use `this.memento`, `getMemento()`, or `asMemento()` to control API
serialization:

```cfml
this.memento = {
    defaultIncludes = [ "id", "firstName", "lastName", "email" ],
    neverInclude = [ "password" ]
};

var payload = user.getMemento();
var page = userEntity
    .asMemento( excludes = [ "modifiedDate" ] )
    .paginate( page = 1, maxRows = 25 );
```

Use `neverInclude` for secrets because callers cannot override it. For read-heavy endpoints that do
not need entity behavior, consider `asQuery()` to return arrays of structs without hydrating Quick
entities.

## Lifecycle events

Entity lifecycle methods use Quick's `pre...` and `post...` names and receive one `eventData`
struct. There are no `beforeCreate`, `afterCreate`, or `beforeDelete` entity hooks.

```cfml
function preInsert( eventData ) {
    // eventData.entity, eventData.builder, eventData.attributes
}

function postInsert( eventData ) {
    // The entity is loaded and its generated key is available here.
    var newId = arguments.eventData.entity.getId();
}

function preUpdate( eventData ) {
    // eventData.entity, eventData.newAttributes, eventData.originalAttributes
}

function preDelete( eventData ) {
    // eventData.entity is the entity about to be deleted.
}
```

Available entity hooks include `instanceReady`, `preLoad`, `postLoad`, `preSave`, `postSave`,
`preInsert`, `postInsert`, `preUpdate`, `postUpdate`, `preDelete`, and `postDelete`. Application
interceptors use the corresponding `quick...` interception point names, such as `quickPostInsert`.
Keep model-local invariants in entity hooks. Put authorization, external I/O, and cross-aggregate
orchestration in the appropriate service or interceptor.

## Soft deletes (Quick 13+)

Enable soft deletes with component metadata and declare the configured timestamp attribute. Calling
`delete()` then performs the soft delete automatically:

```cfml
component
    table="users"
    extends="quick.models.BaseEntity"
    accessors="true"
    softDeletes="true"
{
    property name="id";
    property name="deletedDate" column="deleted_date";
}

user.delete();      // soft delete
user.isTrashed();   // true after soft deletion
user.restore();     // clear the soft-delete timestamp
user.forceDelete(); // permanently delete
```

There is no `user.softDelete()` method. Normal queries exclude soft-deleted rows. Use `withTrashed()`
or `onlyTrashed()` when the caller is authorized to access them.

## Service boundaries

Prefer entity-owned scopes, relationships, and ordinary persistence. Use a service for orchestration,
authorization, transactions, external I/O, or operations spanning multiple aggregates. If a service
only proxies entity queries, inject `quickService:EntityName`; do not invent an `EntityName@quick`
mapping.

## Documentation

- Quick source: https://github.com/coldbox-modules/quick
- Quick guide: https://quick.ortusbooks.com
- qb guide: https://qb.ortusbooks.com
