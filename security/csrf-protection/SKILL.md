---
name: coldbox-security-csrf
description: "Use this skill when implementing CSRF (Cross-Site Request Forgery) protection in ColdBox forms, using cbcsrf to generate and validate tokens, adding csrf() tokens to HTML forms, validating tokens in POST/PUT/DELETE handlers, configuring the cbcsrf module, or excluding API routes from CSRF verification."
applyTo: "**/*.{bx,bxm,cfc,cfm,cfml}"
---

# CSRF Protection in ColdBox

## Overview

CSRF (Cross-Site Request Forgery) attacks trick authenticated users into executing unintended actions. The `cbcsrf` module generates and validates unique per-session tokens for all state-changing requests.

## Language Mode Reference

Examples use **BoxLang (`.bx`)** syntax by default. Adapt for your target language:

| Concept | BoxLang (`.bx`) | CFML (`.cfc`) |
|---------|-----------------|---------------|
| Class declaration | `class [extends="..."] {` | `component [extends="..."] {` |
| DI annotation | `@inject` above `property name="svc";` | `property name="svc" inject="svc";` |
| View templates | `.bxm` suffix | `.cfm` / `.cfml` suffix |
| Tag prefix | `<bx:if>`, `<bx:output>`, `<bx:set>` | `<cfif>`, `<cfoutput>`, `<cfset>` |

> **CFML Compat Mode**: With BoxLang + CFML Compat module, `.bx` and `.cfc` files coexist freely. BoxLang-native classes use `class {}` (`.bx` files); CFML-compat classes use `component {}` (`.cfc` files).

## Installation

```bash
box install cbcsrf
```

## Configuration

```boxlang
// config/ColdBox.cfc
moduleSettings = {
    cbcsrf = {
        // Load an interceptor that verifies all non-GET requests
        enableAutoVerifier     = false,
        // Events to skip verification for, regex allowed: e.g. "stripe\\..*"
        verifyExcludes         = [],
        // Token timeout in minutes, 0 = tokens never expire
        rotationTimeout        = 30,
        // Enable the /cbcsrf/generate endpoint for secured users
        enableEndpoint         = false,
        // WireBox mapping of the token storage
        cacheStorage           = "CacheStorage@cbstorages",
        // Rotate tokens on cbAuth login/logout (cbcsrf default false, cbsecurity default true)
        enableAuthTokenRotator = true
    }
}
```

### With cbsecurity

cbsecurity includes cbcsrf and accepts the same keys under its `csrf` setting. Precedence, highest first:

1. Keys you explicitly set in `cbsecurity.csrf`
2. The `cbcsrf` module settings (your own overrides or its defaults)

Only the keys you set in `cbsecurity.csrf` are applied. cbsecurity defaults never overwrite a `cbcsrf` override.

## Adding CSRF Token to HTML Forms

```html
<form action="#event.buildLink( 'users.store' )#" method="post">
    <!-- Hidden input named "csrf" with a token -->
    #csrf()#

    <div>
        <label>Name: <input type="text" name="name" required /></label>
    </div>

    <button type="submit">Create User</button>
</form>
```

## Verifying Tokens

Set `enableAutoVerifier: true` and the `VerifyCsrf@cbcsrf` interceptor checks every request that is not `GET`, `HEAD` or `OPTIONS`.
The token comes from the `csrf` request value or the `x-csrf-token` header. A missing token throws `TokenNotFoundException`
and an invalid one throws `TokenMismatchException`.

To verify by hand in a handler:

```boxlang
class {

    // POST /users
    function store( event, rc, prc ) {
        if ( !csrfVerify( rc.csrf ?: "" ) ) {
            flash.put( "error", "Invalid security token. Please try again." )
            relocate( "users.create" )
        }

        userService.create( { name: rc.name, email: rc.email } )
        relocate( "users.index" )
    }
}
```

**CFML (`.cfc`):**

```cfml
component {

    function store( event, rc, prc ){
        if ( !csrfVerify( rc.csrf ?: "" ) ) {
            flash.put( "error", "Invalid security token. Please try again." )
            relocate( "users.create" )
        }

        userService.create( { name : rc.name, email : rc.email } )
        relocate( "users.index" )
    }

}
```

## CSRF with AJAX Requests

```html
<head>
    <meta name="csrf-token" content="#csrfToken()#" />
</head>
```

```javascript
fetch( '/users', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'x-csrf-token': document.querySelector( 'meta[name="csrf-token"]' ).content
    },
    body: JSON.stringify( { name: 'John', email: 'john@example.com' } )
} )
```

## Excluding API Routes and Actions

```boxlang
moduleSettings = {
    cbcsrf: {
        enableAutoVerifier: true,
        // Event regex patterns. APIs use JWT/API key auth instead
        verifyExcludes: [
            "^api\\..*",
            "^webhook\\..*"
        ]
    }
}
```

Or annotate a single action with `skipCsrf`:

```boxlang
function webhook( event, rc, prc ) skipCsrf {
    // ...
}
```

## CSRF Token Helpers Reference

| Function | Description |
|----------|-------------|
| `csrf( key, forceNew )` | Hidden `<input name="csrf">` field with a token |
| `csrfField( key, forceNew )` | Same field plus JS that reloads the page when the token expires |
| `csrfToken( key, forceNew )` | Raw token string |
| `csrfVerify( token, key )` | Validate a token, returns boolean |
| `csrfRotate()` | Clear all stored tokens |

## Security Notes

- CSRF protection complements (doesn't replace) authentication
- API routes relying on JWT/API keys don't need CSRF tokens, so exclude them
- Tokens are stored in the configured `cacheStorage` (default `CacheStorage@cbstorages`)
- A short `rotationTimeout` is more secure but may break multi-tab and back-button behavior
- Always use HTTPS so tokens can't be intercepted
