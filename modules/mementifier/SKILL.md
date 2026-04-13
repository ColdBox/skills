---
name: mementifier
description: >
  Use this skill when serializing ColdBox/BoxLang ORM entities or model objects to structs/JSON
  using mementifier. Covers this.memento configuration, getMemento() usage, defaultIncludes/
  defaultExcludes/neverInclude, computed properties via mappers, relationship serialization,
  named profiles, and production API response patterns.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# Mementifier Skill

## When to Use This Skill

Load this skill when:
- Serializing ORM entities or model objects to structs for JSON API responses
- Controlling which properties are exposed (include/exclude allowlists)
- Permanently hiding sensitive fields across all serialization paths
- Computing derived fields at serialization time (e.g., full name, age)
- Creating multiple serialization profiles for different API consumers

## Installation

```bash
box install mementifier
```

## Configuration

### this.memento in an Entity

```js
// models/User.bx (Hibernate entity or WireBox object)
class persistent="true" table="users" {

    property name="id"        fieldtype="id" generator="native";
    property name="firstName" column="first_name";
    property name="lastName"  column="last_name";
    property name="email";
    property name="password";
    property name="role"      default="user";
    property name="createdAt" column="created_at";

    // ---- mementifier config ----
    this.memento = {
        // Listed properties included by default in getMemento()
        defaultIncludes : [
            "id",
            "firstName",
            "lastName",
            "email",
            "role",
            "createdAt",
            "fullName"   // computed — defined via mappers below
        ],

        // Excluded from default output but available in includes override
        defaultExcludes : [],

        // NEVER included in any serialization (passwords, tokens, internal flags)
        neverInclude    : [ "password", "resetToken", "internalFlag" ],

        // Computed/virtual properties that don't map to columns
        mappers : {
            fullName : function( entity ) {
                return trim( entity.getFirstName() & " " & entity.getLastName() )
            },
            initials : function( entity ) {
                return uCase( left( entity.getFirstName(), 1 ) & left( entity.getLastName(), 1 ) )
            }
        }
    }
}
```

## Core Usage

```js
// In handlers or services:
var user    = userService.get( rc.id )
var memento = user.getMemento()      // returns defaultIncludes

// Include extra fields
var memento = user.getMemento( includes = "initials,role" )

// Exclude specific fields
var memento = user.getMemento( excludes = "createdAt" )

// Only specific properties
var memento = user.getMemento( includes = "id,email", excludes = "*" )
```

## Relationships

```js
this.memento = {
    defaultIncludes : [
        "id",
        "title",
        "body",
        "author",       // hasOne relationship
        "tags"          // hasMany relationship
    ],

    // Specify relationship memento depth
    defaultRelationships : {
        // Relationships are called via getPropertyName()
        // mementifier calls getMemento() on related entities
    }
}
```

### Serializing Relations Manually (for custom depth control)

```js
// In the mapper, serialize author with limited fields
mappers : {
    authorSummary : function( entity ) {
        var author = entity.getAuthor()
        return isNull( author ) ? {} : author.getMemento( includes = "id,firstName,lastName" )
    },
    tagList : function( entity ) {
        return entity.getTags().map( ( tag ) => tag.getMemento( includes = "id,name" ) )
    }
}
```

## Named Profiles

```js
this.memento = {
    defaultIncludes : [ "id", "firstName", "email" ],
    neverInclude    : [ "password" ],

    // Named profiles for different audiences
    profiles : {
        admin : {
            includes : [ "id", "firstName", "lastName", "email", "role", "createdAt", "internalNotes" ]
        },
        public : {
            includes : [ "id", "firstName" ]
        },
        api : {
            includes : [ "id", "firstName", "lastName", "email", "role" ]
        }
    }
}
```

```js
// Use a profile
user.getMemento( profile = "admin" )
user.getMemento( profile = "public" )
```

## Production Patterns

### Handler — REST API Serialization

```js
function show( event, rc, prc ) {
    var user = userService.getOrFail( rc.id )
    event.renderData( type = "json", data = user.getMemento() )
}

function index( event, rc, prc ) {
    var users = userService.list()

    // Serialize array of entities
    event.renderData(
        type = "json",
        data = users.map( ( user ) => user.getMemento() )
    )
}
```

### Admin vs Public API Profiles

```js
function show( event, rc, prc ) {
    var user    = userService.getOrFail( rc.id )
    var profile = security.hasRole( "admin" ) ? "admin" : "public"

    event.renderData( type = "json", data = user.getMemento( profile = profile ) )
}
```

### Nested Mementos

```js
// Order with line items
function show( event, rc, prc ) {
    var order = orderService.getOrFail( rc.id )

    var data = order.getMemento()
    data.items = order.getItems().map( ( item ) => item.getMemento( includes = "id,productName,qty,price" ) )

    event.renderData( type = "json", data = data )
}
```

## Best Practices

- **Always populate `neverInclude`** for sensitive fields — passwords, tokens, API keys, SSNs
- **Use `defaultIncludes`** to define the safe default surface — opt-in rather than opt-out
- **Use profiles** when the same entity serves multiple audiences (public API vs admin)
- **Define mappers for computed fields** — keeps business derivation logic centralized
- **Never expose `neverInclude` fields** even if the caller passes them in `includes` — mementifier enforces this
- **Keep mementos flat** unless nesting is required — deeply nested responses are harder to consume
- **Test your mementos** to confirm sensitive fields are absent from serialized output

## Documentation

- mementifier: https://github.com/coldbox-modules/mementifier
