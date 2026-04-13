---
name: cors
description: >
  Use this skill when configuring Cross-Origin Resource Sharing (CORS) for ColdBox/BoxLang REST APIs
  using the cors module. Covers allowed origins, methods, headers, credentials, preflight OPTIONS
  handling, dynamic origin functions, and production hardening.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# CORS Skill

## When to Use This Skill

Load this skill when:
- Enabling cross-origin requests to a ColdBox REST API from a browser-based frontend
- Restricting allowed origins to a specific whitelist in production
- Allowing credentials (cookies, Authorization headers) in cross-origin requests
- Handling preflight OPTIONS requests automatically
- Dynamically computing allowed origins based on request context

## Installation

```bash
box install cors
```

## Configuration

### config/modules/cors.cfc

```js
function configure() {
    return {
        // List of allowed origins. Use "*" to allow all (not recommended for credentialed requests)
        allowOrigins     : getSystemSetting( "CORS_ORIGINS", "*" ),

        // Allowed HTTP methods
        allowMethods     : "GET,POST,PUT,PATCH,DELETE,OPTIONS",

        // Headers the browser is allowed to send
        allowHeaders     : "Content-Type,Authorization,X-Requested-With,X-CSRF-Token",

        // Headers the browser is allowed to read from the response
        exposeHeaders    : "X-Total-Count,X-Page-Size,X-Page",

        // Allow cookies / Authorization headers in CORS requests
        allowCredentials : false,

        // How long browsers may cache the preflight response (seconds)
        preflightMaxAge  : 3600
    }
}
```

## Multiple Allowed Origins

```js
function configure() {
    return {
        allowOrigins : [
            "https://app.example.com",
            "https://admin.example.com",
            "https://staging.example.com"
        ],
        allowMethods     : "GET,POST,PUT,DELETE,OPTIONS",
        allowHeaders     : "Content-Type,Authorization",
        allowCredentials : true   // required when frontend sends cookies
    }
}
```

## Dynamic Origin Validation

```js
function configure() {
    return {
        // Function receives current request origin; return true to allow
        allowOrigins : function( origin ) {
            var allowed = [
                "https://app.example.com",
                "https://admin.example.com"
            ]

            // Allow any subdomain of example.com in development
            if ( getSystemSetting( "ENVIRONMENT", "production" ) == "development" ) {
                return reFind( "^https?://.*\.example\.com(:\d+)?$", origin )
            }

            return allowed.find( origin ) > 0
        },
        allowCredentials : true,
        allowMethods     : "GET,POST,PUT,DELETE,OPTIONS"
    }
}
```

## Production Patterns

### API-Only Express Configuration

```js
// Strict allowlist — production REST API consumed by known frontends
function configure() {
    return {
        allowOrigins     : [
            "https://app.example.com",
            "https://mobile.example.com"
        ],
        allowMethods     : "GET,POST,PUT,DELETE",
        allowHeaders     : "Content-Type,Authorization,X-CSRF-Token",
        exposeHeaders    : "X-Total-Count",
        allowCredentials : true,
        preflightMaxAge  : 7200     // 2 hours
    }
}
```

### Configuration with Different Environments

```js
function configure() {
    var env = getSystemSetting( "ENVIRONMENT", "production" )

    return {
        allowOrigins : ( env == "development" )
            ? "*"
            : [ "https://app.example.com" ],
        allowMethods     : "GET,POST,PUT,PATCH,DELETE,OPTIONS",
        allowHeaders     : "Content-Type,Authorization",
        allowCredentials : ( env != "development" )
    }
}
```

## Best Practices

- **Never use `"*"` in production with `allowCredentials: true`** — browsers block this combination
- **Enumerate exact origins** — avoid wildcard patterns in production allowlists
- **Use `allowCredentials: true`** only when the frontend must send cookies or Authorization headers
- **Set a reasonable `preflightMaxAge`** — reduces OPTIONS round-trips without sacrificing flexibility
- **Expose only necessary response headers** via `exposeHeaders` — keep the surface minimal
- **Do not rely on CORS as a security layer** — it is a browser courtesy; validate all API tokens server-side
- **Test preflight handling** explicitly — confirm OPTIONS `/api/*` returns correct headers

## Documentation

- cors module: https://github.com/coldbox-modules/cors
- MDN CORS guide: https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS
