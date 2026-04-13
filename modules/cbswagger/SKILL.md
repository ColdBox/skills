---
name: cbswagger
description: >
  Use this skill when generating OpenAPI 3.x (Swagger) documentation for ColdBox/BoxLang REST APIs
  using cbswagger. Covers installation, module configuration, handler/action JSDoc annotations,
  request/response schemas, security definitions, parameter documentation, and accessing the
  generated spec.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# CBSwagger Skill

## When to Use This Skill

Load this skill when:
- Auto-generating OpenAPI 3.x documentation from ColdBox handler annotations
- Documenting REST API endpoints, parameters, request bodies, and responses
- Adding JWT/Bearer security definitions to the generated spec
- Configuring the Swagger UI endpoint for developer exploration
- Integrating API documentation into a CI/CD pipeline

## Installation

```bash
box install cbswagger
```

## Configuration

### config/modules/cbswagger.cfc

```js
function configure() {
    return {
        // Path to output the generated JSON spec
        jsonPath     : "/includes/spec.json",
        // Swagger UI route
        swaggerRoute : "/api/openapi",

        // OpenAPI info block
        info : {
            title       : "My API",
            description : "REST API documentation",
            version     : "1.0.0",
            contact     : {
                name  : "API Support",
                email : "api@example.com",
                url   : "https://example.com/support"
            }
        },

        // Servers list
        servers : [
            { url: "https://api.example.com", description: "Production" },
            { url: "http://localhost:8500",   description: "Development" }
        ],

        // Security schemes
        securityDefinitions : {
            BearerAuth : {
                type         : "http",
                scheme       : "bearer",
                bearerFormat : "JWT"
            }
        },

        // Apply security globally
        defaultSecurity : [ { BearerAuth: [] } ],

        // Handler paths to scan for routes
        handlerPaths : [ "handlers" ]
    }
}
```

## Annotating Handlers

### Class-Level Annotation

```js
/**
 * @tag         Users
 * @description CRUD operations on the User resource
 */
@secured
class UsersHandler extends coldbox.system.EventHandler {
```

### Action-Level Annotations

```js
/**
 * @summary     List all users
 * @description Returns a paginated list of users. Requires admin role.
 * @response    200 { schema: "User", isArray: true }
 * @response    401 Unauthorized
 * @response    403 Forbidden
 * @param       page  { in: "query", type: "integer", description: "Page number", default: 1 }
 * @param       limit { in: "query", type: "integer", description: "Items per page", default: 25 }
 */
function index( event, rc, prc ) {
    // ...
}

/**
 * @summary     Get user by ID
 * @description Returns a single user by their UUID primary key
 * @response    200 { schema: "User" }
 * @response    404 Not Found
 * @param       id { in: "path", type: "string", format: "uuid", required: true }
 */
function show( event, rc, prc ) {
    // ...
}

/**
 * @summary     Create user
 * @description Creates a new user account
 * @requestBody { schema: "UserCreate", required: true }
 * @response    201 { schema: "User" }
 * @response    422 { schema: "ValidationError" }
 */
function create( event, rc, prc ) {
    // ...
}

/**
 * @summary     Update user
 * @requestBody { schema: "UserUpdate" }
 * @response    200 { schema: "User" }
 * @response    404 Not Found
 * @response    422 { schema: "ValidationError" }
 * @param       id { in: "path", type: "string", format: "uuid", required: true }
 */
function update( event, rc, prc ) {
    // ...
}

/**
 * @summary  Delete user
 * @response 204 No Content
 * @response 404 Not Found
 * @param    id { in: "path", type: "string", format: "uuid", required: true }
 */
function delete( event, rc, prc ) {
    // ...
}
```

## Schema Files

Place JSON schema files in `models/schemas/` (or configure a different path):

### models/schemas/User.json

```json
{
    "type": "object",
    "properties": {
        "id":        { "type": "string", "format": "uuid" },
        "name":      { "type": "string" },
        "email":     { "type": "string", "format": "email" },
        "createdAt": { "type": "string", "format": "date-time" }
    },
    "required": ["id", "name", "email"]
}
```

### models/schemas/ValidationError.json

```json
{
    "type": "object",
    "properties": {
        "message": { "type": "string" },
        "errors": {
            "type": "array",
            "items": {
                "type": "object",
                "properties": {
                    "field":   { "type": "string" },
                    "message": { "type": "string" }
                }
            }
        }
    }
}
```

## Accessing the Spec

- **JSON spec**: `GET /api/openapi` (returns raw OpenAPI JSON)
- **Swagger UI**: `GET /api/openapi/ui` (interactive browser)
- **ReDoc UI**: `GET /api/openapi/redoc`

## Best Practices

- **Annotate every public action** — undocumented endpoints create consumer confusion
- **Use `@tag` grouping** — organizes large APIs into logical sections in the UI
- **Define shared schemas** in JSON files — avoid duplicating inline schema structs
- **Document all possible response codes** — include 401, 404, 422 even if the framework handles them
- **Protect the Swagger UI in production** — restrict to internal IPs or require authentication
- **Regenerate the spec in CI** — commit the spec file and detect drift from annotations
- **Use semantic versioning** in `info.version` — helps consumers track breaking changes

## Documentation

- cbswagger: https://github.com/coldbox-modules/cbswagger
- OpenAPI 3.0 spec: https://swagger.io/specification/
