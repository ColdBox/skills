---
name: coldbox-routing-development
description: "Use this skill when configuring ColdBox routes, setting up RESTful resource routes, creating route groups, implementing URL pattern matching with constraints, defining named routes, answering a route inline with a toResponse() closure, exposing BoxLang AI surfaces with the toAi(), toMCP() and toAiGateway() route terminators, or working with Router.cfc in a ColdBox application."
applyTo: "**/*.{bx,bxm,cfc,cfm,cfml}"
---

# Routing Development

## When to Use This Skill

Use this skill when setting up URL routing for ColdBox applications, defining REST resource routes, or configuring route constraints and groups.

## Language Mode Reference

Examples use **BoxLang (`.bx`)** syntax by default. Adapt for your target language:

| Concept | BoxLang (`.bx`) | CFML (`.cfc`) |
|---------|-----------------|---------------|
| Class declaration | `class [extends="..."] {` | `component [extends="..."] {` |
| DI annotation | `@inject` above `property name="svc";` | `property name="svc" inject="svc";` |
| View templates | `.bxm` suffix | `.cfm` / `.cfml` suffix |
| Tag prefix | `<bx:if>`, `<bx:output>`, `<bx:set>` | `<cfif>`, `<cfoutput>`, `<cfset>` |

> **CFML Compat Mode**: With BoxLang + CFML Compat module, `.bx` and `.cfc` files coexist freely. BoxLang-native classes use `class {}` (`.bx` files); CFML-compat classes use `component {}` (`.cfc` files).

## Core Concepts

ColdBox routing maps URLs to handler actions via `config/Router.cfc`:
- RESTful resource routes are defined via `resources()` or `route()`
- Route groups share prefix, namespace, or middleware
- Constraints validate URL segments with regex
- Named routes can be used in views/handlers via `buildLink()`
- HTTP verb restrictions enforce RESTful semantics

## Implementation Steps

1. Create/open `config/Router.cfc`
2. Define resource routes or individual routes
3. Add route groups where appropriate
4. Apply constraints to dynamic segments
5. Name routes for reference in templates
6. Order routes from most-specific to least-specific

## Basic Router.cfc

```boxlang
class Router extends coldbox.system.web.routing.Router {

    function configure() {

        // Set base URL and options
        setFullRewrites( true )

        // Simple home route
        route( "/", "main.index" )

        // Named route
        route( name = "about", pattern = "/about", target = "pages.about" )

        // RESTful resource (generates CRUD routes)
        resources( "users" )

        // Nested resources
        resources(
            resource  = "posts",
            nested    = "comments"
        )

        // Wildcard route - MUST be last
        route( "/:handler/:action?" )
    }
}
```

**CFML (`.cfc`):**

```cfml
component extends="coldbox.system.web.routing.Router" {

    function configure() {

        // Set base URL and options
        setFullRewrites( true )

        // Simple home route
        route( "/", "main.index" )

        // Named route
        route( name = "about", pattern = "/about", target = "pages.about" )

        // RESTful resource (generates CRUD routes)
        resources( "users" )

        // Nested resources
        resources(
            resource  = "posts",
            nested    = "comments"
        )

        // Wildcard route - MUST be last
        route( "/:handler/:action?" )
    }
}
```

## RESTful Resource Routes

`resources( "users" )` generates these routes:

| Method   | URL             | Handler Action  |
|----------|-----------------|-----------------|
| GET      | /users          | users.index     |
| GET      | /users/new      | users.create    |
| POST     | /users          | users.store     |
| GET      | /users/:id      | users.show      |
| GET      | /users/:id/edit | users.edit      |
| PUT      | /users/:id      | users.update    |
| PATCH    | /users/:id      | users.update    |
| DELETE   | /users/:id      | users.delete    |

## Route Groups

```boxlang
// API versioning group
group(
    pattern   = "/api/v1",
    namespace = "api.v1",
    handler   = "api.v1"
) {
    resources( "users" )
    resources( "posts" )
    resources( "comments" )
}

// Admin group with prefix
group(
    pattern = "/admin",
    handler = "admin"
) {
    route( "/", "admin.dashboard.index" )
    resources( "users" )
    resources( "settings" )
}

// Authenticated group (with CBSecurity middleware)
group(
    pattern    = "/dashboard",
    middleware = "cbsecurity"
) {
    route( "/", "dashboard.index" )
    route( "/profile", "dashboard.profile" )
}
```

## Routes with Constraints

```boxlang
// Numeric ID constraint
route(
    pattern     = "/users/:id",
    target      = "users.show",
    constraints = { id: "[0-9]+" }
)

// UUID constraint
route(
    pattern     = "/tokens/:token",
    target      = "tokens.verify",
    constraints = { token: "[a-f0-9]{8}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{12}" }
)

// Slug constraint
route(
    pattern     = "/blog/:slug",
    target      = "blog.show",
    constraints = { slug: "[a-z0-9-]+" }
)

// Optional segments
route( "/search/:term?", "search.index" )
```

## HTTP Verb-Specific Routes

```boxlang
// Explicit HTTP methods
get( "/users", "users.index" )
post( "/users", "users.store" )
put( "/users/:id", "users.update" )
patch( "/users/:id", "users.patch" )
delete( "/users/:id", "users.delete" )

// Multiple verbs
route(
    pattern = "/users/:id",
    target  = "users.show"
).methods( "GET,HEAD" )
```

## Named Routes in Views

```boxlang
// In handler
var userLink = buildLink( "users.show", { id: prc.user.getId() } )

// In view template
<a href="#buildLink( 'users.index' )#">All Users</a>
<a href="#buildLink( 'users.edit', { id: prc.user.getId() } )#">Edit</a>

// Or using named route
<a href="#buildLink( routeName = 'user.profile', queryString = { tab: 'settings' } )#">Profile</a>
```

## API Router (Dedicated)

```boxlang
// config/Router.cfc
class Router extends coldbox.system.web.routing.Router {

    function configure() {
        setFullRewrites( true )

        // Mount API routes from module
        addRoute( route( "/api/" ).toModuleRoutes( "api" ) )

        // Web routes
        route( "/", "main.index" )
        route( "/:handler/:action?" )
    }
}

// modules/api/config/Router.cfc
class Router extends coldbox.system.web.routing.Router {

    function configure() {
        // V1 routes
        group( pattern = "/v1" ) {
            resources( "users" )
            resources( "posts" )
        }

        // V2 routes
        group(
            pattern   = "/v2",
            namespace = "v2"
        ) {
            resources( "users" )
        }
    }
}
```

**CFML (`.cfc`):**

```cfml
// config/Router.cfc
component extends="coldbox.system.web.routing.Router" {

    function configure() {
        setFullRewrites( true )

        // Mount API routes from module
        addRoute( route( "/api/" ).toModuleRoutes( "api" ) )

        // Web routes
        route( "/", "main.index" )
        route( "/:handler/:action?" )
    }
}

// modules/api/config/Router.cfc
component extends="coldbox.system.web.routing.Router" {

    function configure() {
        // V1 routes
        group( pattern = "/v1" ) {
            resources( "users" )
            resources( "posts" )
        }

        // V2 routes
        group(
            pattern   = "/v2",
            namespace = "v2"
        ) {
            resources( "users" )
        }
    }
}
```

## Inline Route Responses — `toResponse()`

Terminate a route with a closure (or a plain string) instead of a handler action. Useful for
health checks, webhooks, and small endpoints that do not earn a handler:

```boxlang
// Static body + status
route( "/health" ).toResponse( "OK" )
route( "/gone" ).toResponse( "Removed", 410 )

// Closure receives ( event, rc, prc )
route( "/api/ping" ).toResponse( ( event, rc, prc ) => {
    return { "pong": true, "at": now() }
} )
```

The closure's return value is serialized for you. A closure may also **render the response
itself** with `event.renderData()` when it needs per-request control over status or content type:

```boxlang
route( "/api/orders/:id" ).toResponse( ( event, rc, prc ) => {
    var order = getInstance( "OrderService" ).find( rc.id )

    if ( isNull( order ) ) {
        return event.renderData( type: "json", data: { "error": "Not found" }, statusCode: 404 )
    }

    return event.renderData( type: "json", data: order, statusCode: 200 )
} )
```

> **ColdBox 8.2.0 fix:** a closure that called `event.renderData()` previously had its status code
> and content type flattened back onto the route's own static `statusCode` by the router's
> follow-up `renderData()` call, so every request answered with the same status no matter what the
> closure set. Render data set *inside* the closure is now left alone, so a closure can answer
> `404` on one request and `200` on the next. Render data an interceptor set **before** the route
> ran is unaffected. On 8.1 and earlier, route to a handler action instead when you need a
> per-request status.

---

## BoxLang AI Route Terminators

> **BoxLang only.** `toAi()`, `toMCP()` and `toAiGateway()` all require BoxLang and the `bxai`
> module; they throw at route-registration time on any other runtime.

| Terminator | Exposes |
|---|---|
| `toAi( ... )` | A chat/completion endpoint over HTTP |
| `toMCP( ... )` | A Model Context Protocol server |
| `toAiGateway( [gateway], [session] )` | A BoxLang AI Gateway — platform webhooks, human-in-the-loop approvals, and gateway info (**ColdBox 8.2.0+**) |

### `toAiGateway()` — Mounting an AI Gateway over HTTP

*ColdBox 8.2.0+.* One call registers the whole sub-route family a gateway surface needs:

| Verb | Pattern | Purpose |
|---|---|---|
| `GET`/`POST` | `{pattern}[/:gateway]/events` | The platform's URL-verification handshake (GET) and its inbound events (POST) |
| `GET` | `{pattern}/interactions/:requestID` | Poll a pending human-in-the-loop interaction |
| `POST` | `{pattern}/interactions/:requestID/decisions` | Submit a human's decision |
| `GET` | `{pattern}/info` | What this mount serves and which gateways are behind it |

GET and POST deliberately share the `/events` path: a platform is given **one** URL to store and
verifies it with a GET before it ever POSTs to it.

```boxlang
// config/Router.cfc
class extends="coldbox.system.web.routing.Router" {

    function configure() {

        // One mount serving EVERY gateway in aiGatewayRegistry().
        // The terminator inserts its own :gateway placeholder, so the platform's
        // name arrives in the URL: POST /gateways/slack/events
        route( "/gateways" ).toAiGateway( session: "SupportAgentSession" )

        // Pinned to a single gateway — no placeholder segment:
        // POST /webhooks/slack/events
        route( "/webhooks/slack" ).toAiGateway( "slack", "SupportAgentSession" )

        // Verify and parse only: inbound events are validated and normalized,
        // and handed back for the application to dispatch itself.
        route( "/gateways" ).toAiGateway()

    }

}
```

**The `session` argument is what makes it an agent.** Pass a WireBox ID or a live `GatewaySession`
and every inbound message is dispatched as an agent turn and acked `202` **immediately**, without
waiting for the turn to finish — a platform webhook times out in seconds while an agent turn does
not. The response reports which thread each message landed on, so the caller can correlate the
reply that arrives later. Omit it and the route verifies and parses only, dispatching nothing.

```boxlang
// A live session instead of a WireBox ID
route( "/gateways" ).toAiGateway( session: aiGatewaySession( "support" ) )
```

Route modifiers already set on the fluent chain (`withDomain`, `withSSL`, `withCondition`,
`header`, and so on) are inherited by all the sub-routes:

```boxlang
route( "/gateways" )
    .withSSL()
    .withDomain( "hooks.example.com" )
    .toAiGateway( session: "SupportAgentSession" )
```

Each sub-route is named off the mount, so you can build links to them:

```boxlang
buildLink( "gateways.gateway.info" )          // GET  /gateways/info
buildLink( "gateways.gateway.events" )        // GET/POST /gateways/:gateway/events
buildLink( "gateways.gateway.interaction" )   // GET  /gateways/interactions/:requestID
buildLink( "gateways.gateway.decision" )      // POST /gateways/interactions/:requestID/decisions
```

> The base name is the route's `name` if you set one, otherwise its pattern. Passing anything but
> a WireBox ID string or an object as `session` throws `InvalidArgumentException` at registration.

See the [`coldbox-ai-integration`](../ai-integration/SKILL.md) skill for building the agent and
session that sit behind the mount.

---

## Route Best Practices

- Define most-specific routes first, wildcards last
- Use `resources()` for standard CRUD routes rather than defining each route manually
- Group related routes with shared prefixes/namespaces
- Add constraints for numeric IDs and UUIDs to prevent invalid parameters
- Name important routes for easy reference in templates
- Use HTTP method restrictions for REST APIs
- Separate API routing via modules for cleaner organization
- Use `toResponse()` for trivial endpoints, but move to a handler once there is logic to test
- Mount AI gateways with `toAiGateway()` rather than hand-registering the five sub-routes
