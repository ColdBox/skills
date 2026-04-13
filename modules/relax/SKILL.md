---
name: relax
description: >
  Use this skill when modeling REST APIs with the Relax DSL, defining routes with parameters and
  response schemas, generating API documentation, and producing JSON/XML specification output
  for ColdBox-based REST services.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# Relax — REST API Documentation Skill

## When to Use This Skill

Load this skill when:
- Documenting a ColdBox REST API with Relax DSL
- Defining route parameters, body schemas, and response models
- Generating JSON/XML API specification output
- Producing browsable API documentation from a Relax model
- Exporting API specs for consumption by API gateways or client generators

## Installation

```bash
box install relax
```

## Core Concepts

Relax is an API modeling DSL that allows you to describe your REST API using a fluent CFC-based syntax. The model is used to generate documentation and exportable specs.

## Creating a Relax API Model

```js
// models/MyAPIRelax.bx (BoxLang)
class extends="relax.models.RelaxDSL" {

    function configure() {

        // API metadata
        this.relax = {
            title: "My Application API",
            version: "v1",
            description: "ColdBox REST API documentation",
            contact: "devteam@example.com",
            license: "MIT",
            basePath: "/api/v1",
            protocols: [ "https" ]
        }

        // Global request headers
        globalHeader(
            name: "Authorization",
            description: "Bearer token for authentication",
            required: true,
            type: "string"
        )

        // Define resources
        api()
    }

    private function api() {

        // =========================================================
        // Users
        // =========================================================

        resource(
            pattern: "/users",
            description: "User management"
        )
        .verb(
            verb: "GET",
            description: "List all users",
            response: { "200": "UserListResponse" },
            parameters: [
                { name: "page", type: "integer", required: false, description: "Page number" },
                { name: "pageSize", type: "integer", required: false, description: "Items per page" }
            ]
        )
        .verb(
            verb: "POST",
            description: "Create a new user",
            body: "UserCreateRequest",
            response: { "201": "UserResponse", "422": "ValidationError" }
        )

        resource(
            pattern: "/users/:userID",
            description: "Individual user operations"
        )
        .verb(
            verb: "GET",
            description: "Retrieve a user by ID",
            response: { "200": "UserResponse", "404": "NotFound" },
            parameters: [
                { name: "userID", type: "integer", required: true, description: "User ID" }
            ]
        )
        .verb(
            verb: "PUT",
            description: "Update a user",
            body: "UserUpdateRequest",
            response: { "200": "UserResponse", "404": "NotFound", "422": "ValidationError" }
        )
        .verb(
            verb: "DELETE",
            description: "Delete a user",
            response: { "204": "NoContent", "404": "NotFound" }
        )
    }
}
```

## CFML Syntax

```cfscript
// models/MyAPIRelax.cfc
component extends="relax.models.RelaxDSL" {

    function configure() {
        this.relax = {
            title = "My Application API",
            version = "v1",
            basePath = "/api/v1"
        }
    }
}
```

## Generating Documentation

### Via ColdBox Handler

```js
// handlers/APIDocs.bx
class extends="coldbox.system.EventHandler" {

    property name="relaxService" inject="RelaxService@relax"

    function index( event, rc, prc ) {
        prc.apiModel = relaxService.getAPIModel( "MyAPIRelax" )
        event.setView( "apidocs/index" )
    }

    // Export JSON spec
    function exportJSON( event, rc, prc ) {
        var spec = relaxService.exportJSON( "MyAPIRelax" )
        event.renderData( type="json", data=spec )
    }

    // Export XML spec
    function exportXML( event, rc, prc ) {
        var spec = relaxService.exportXML( "MyAPIRelax" )
        event.renderData( type="xml", data=spec )
    }
}
```

### Routing

```js
// config/Router.bx
router
    .route( "/api-docs" ).to( "APIDocs.index" )
    .route( "/api-docs/export.json" ).to( "APIDocs.exportJSON" )
    .route( "/api-docs/export.xml" ).to( "APIDocs.exportXML" )
```

## Schema Definitions

```js
// Define response schemas in the configure() method
schema( name: "UserResponse", properties: {
    id: { type: "integer" },
    name: { type: "string" },
    email: { type: "string", format: "email" },
    createdAt: { type: "string", format: "date-time" }
} )

schema( name: "ValidationError", properties: {
    error: { type: "string" },
    messages: { type: "array", items: { type: "string" } }
} )
```

## Production Patterns

### Restrict Docs to Non-Production

```js
// handlers/APIDocs.bx
class extends="coldbox.system.EventHandler" {

    function preHandler( event, rc, prc, action, eventArguments ) {
        if ( getSetting( "environment" ) == "production" ) {
            relocate( url="/" )
        }
    }
}
```

### Auto-Register Model via WireBox

```js
// config/WireBox.bx
class extends="wirebox.system.ioc.config.Binder" {
    function configure() {
        map( "MyAPIRelax" ).to( "models.MyAPIRelax" ).asSingleton()
    }
}
```

## Best Practices

- **Keep Relax models separate from handlers** — place in `models/` or `models/api/` folder
- **Use schemas** to document request/response bodies — avoids repeating structure per verb
- **Restrict API docs to non-production** — document endpoints should not be publicly accessible in prod
- **Version your API** in `basePath` (e.g., `/api/v1`) — enables clean versioning
- **Document error responses** explicitly per verb — essential for API consumers
- **Export JSON spec** for use with API gateways and client generators (Swagger UI, Postman)
- **Always define `title`, `version`, and `basePath`** in `this.relax` — required for valid spec output

## Documentation

- Relax: https://forgebox.io/view/relax
- Relax GitHub: https://github.com/coldbox-modules/relax
