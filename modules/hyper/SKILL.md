---
name: hyper
description: >
  Use this skill when making HTTP requests from ColdBox/BoxLang applications using the Hyper HTTP
  client. Covers HyperBuilder injection, fluent request construction, GET/POST/PUT/DELETE shortcuts,
  headers, authentication, timeout configuration, response handling, error handling, and production
  patterns for REST API integrations.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# Hyper Skill

## When to Use This Skill

Load this skill when:
- Making HTTP requests to external REST APIs or services
- Building a reusable HTTP client service around a third-party API
- Handling JSON, form, or multipart request bodies
- Authenticating with Bearer tokens, Basic auth, or API keys
- Implementing retry logic, timeouts, and error handling for HTTP calls

## Installation

```bash
box install hyper
```

## Core API

### Injection

```js
property name="hyper" inject="HyperBuilder@hyper";
```

### Request Shortcuts

```js
// GET
var response = hyper.get( "https://api.example.com/users" )

// POST with JSON body
var response = hyper.post( "https://api.example.com/users", { name: "Alice", email: "alice@example.com" } )

// PUT
var response = hyper.put( "https://api.example.com/users/1", { name: "Alice Updated" } )

// PATCH
var response = hyper.patch( "https://api.example.com/users/1", { email: "new@example.com" } )

// DELETE
var response = hyper.delete( "https://api.example.com/users/1" )
```

### Fluent Builder

```js
var response = hyper
    .new()
    .setBaseURL( "https://api.example.com" )
    .setURL( "/users" )
    .withHeaders( {
        "Authorization" : "Bearer #token#",
        "Accept"        : "application/json"
    } )
    .setQueryParams( { page: 1, limit: 25 } )
    .setTimeout( 30 )
    .get()
```

### Request Body Types

```js
// JSON body (default when passing a struct to post()/put())
hyper.post( url, { key: "value" } )

// Form body (x-www-form-urlencoded)
hyper.new()
    .asFormParams()
    .setBody( { username: "alice", password: "secret" } )
    .post( "https://auth.example.com/token" )

// Raw body
hyper.new()
    .withHeader( "Content-Type", "text/plain" )
    .setBody( "raw content here" )
    .post( "https://api.example.com/webhook" )
```

### Response API

```js
var response = hyper.get( "https://api.example.com/users" )

response.getStatusCode()        // 200
response.isSuccess()            // true if 2xx
response.isClientError()        // true if 4xx
response.isServerError()        // true if 5xx
response.getBody()              // raw response string
response.json()                 // deserialized struct/array
response.getHeader( "Content-Type" )
response.getHeaders()           // all response headers
```

## Production Patterns

### Reusable API Client Service

```js
// services/StripeService.bx
class {

    property name="hyper" inject="HyperBuilder@hyper";

    private string function getBaseURL() {
        return "https://api.stripe.com/v1"
    }

    private HyperRequest function newRequest() {
        return hyper
            .new()
            .setBaseURL( getBaseURL() )
            .withHeader( "Authorization", "Bearer #getSystemSetting( 'STRIPE_SECRET_KEY' )#" )
            .setTimeout( 30 )
    }

    struct function createCustomer( email, name ) {
        var response = newRequest()
            .asFormParams()
            .setBody( { email: email, name: name } )
            .post( "/customers" )

        if ( !response.isSuccess() ) {
            throw(
                type    = "StripeAPIException",
                message = "Failed to create Stripe customer: #response.json().error.message#"
            )
        }

        return response.json()
    }

    struct function getCustomer( customerId ) {
        var response = newRequest().get( "/customers/#customerId#" )

        if ( response.getStatusCode() == 404 ) {
            throw( type = "StripeCustomerNotFound", message = "Customer #customerId# not found" )
        }

        if ( !response.isSuccess() ) {
            throw( type = "StripeAPIException", message = response.getBody() )
        }

        return response.json()
    }
}
```

### Token Caching Pattern

```js
class {

    property name="hyper"        inject="HyperBuilder@hyper";
    property name="cacheStorage" inject="CacheStorage@cbstorages";

    private string function getAccessToken() {
        var cacheKey = "oauth_access_token"

        if ( cacheStorage.exists( cacheKey ) ) {
            return cacheStorage.get( cacheKey )
        }

        var response = hyper.new()
            .asFormParams()
            .setBody( {
                grant_type    : "client_credentials",
                client_id     : getSystemSetting( "API_CLIENT_ID" ),
                client_secret : getSystemSetting( "API_CLIENT_SECRET" )
            } )
            .post( "https://auth.example.com/oauth/token" )

        if ( !response.isSuccess() ) {
            throw( type = "AuthException", message = "Failed to obtain access token" )
        }

        var token = response.json().access_token

        // Cache for 50 minutes (token expires in 60)
        cacheStorage.set( cacheKey, token, 50 )

        return token
    }
}
```

### Handling 4xx/5xx Errors

```js
function callApi( endpoint ) {
    var response = hyper.get( "https://api.example.com" & endpoint )

    switch ( response.getStatusCode() ) {
        case 200: return response.json()
        case 401: throw( type = "UnauthorizedException",   message = "API token invalid or expired" )
        case 403: throw( type = "ForbiddenException",      message = "API access denied" )
        case 404: throw( type = "NotFoundException",       message = "Resource not found: #endpoint#" )
        case 429: throw( type = "RateLimitException",      message = "API rate limit exceeded" )
        default:
            throw(
                type    = "APIException",
                message = "API error #response.getStatusCode()#: #response.getBody()#"
            )
    }
}
```

## Best Practices

- **Never hardcode API keys** — always use `getSystemSetting()` or environment variables
- **Set timeouts** (`setTimeout()`) on every request — unbounded HTTP calls can stall server threads
- **Check `isSuccess()`** before accessing `json()` — calling `json()` on a 4xx/5xx body may fail
- **Build reusable client services** rather than calling `hyper` directly from handlers
- **Cache OAuth tokens** near their TTL — avoids unnecessary token-request overhead
- **Throw typed exceptions** from API wrapper methods — handlers can catch specific failure modes
- **Log HTTP errors** with enough context (URL, status, body snippet) for debugging
- **Use `asFormParams()`** for OAuth token endpoints — many identity providers require form encoding

## Documentation

- hyper: https://github.com/coldbox-modules/hyper
- hyper docs: https://hyper.ortusbooks.com
