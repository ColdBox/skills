---
name: coldbox-security-csrf
description: "Use this skill when implementing CSRF (Cross-Site Request Forgery) protection in ColdBox forms, using cbcsrf to generate and validate tokens, adding csrf() tokens to HTML forms, validating tokens in POST/PUT/DELETE handlers, configuring the cbcsrf module, or excluding API routes from CSRF verification."
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
    cbcsrf: {
        enabled: true,
        tokenKey: "_csrftoken",

        // Rotate token for each request (more secure, may cause issues with multi-tab)
        rotateTokens: false,

        // Which HTTP methods require verification
        verifyMethod: "all",  // or "post", "delete", "put", "patch"

        // Token expiration in minutes
        tokenExpiration: 30,

        // Exclude these event/route patterns from CSRF
        exclude: [
            "^api\\..*"   // exclude all API routes
        ]
    }
}
```

## Adding CSRF Token to HTML Forms

```html
<!-- views/users/create.cfm -->
<form action="#event.buildLink( 'users.store' )#" method="post">

    <!-- Drop the CSRF token field — auto-generates the hidden input -->
    #csrf()#

    <div>
        <label>Name: <input type="text" name="name" required /></label>
    </div>
    <div>
        <label>Email: <input type="email" name="email" required /></label>
    </div>

    <button type="submit">Create User</button>
</form>
```

## Manual Token Generation

```boxlang
// In handler — pass token to view
function create( event, rc, prc ) {
    prc.csrfToken = generateCSRFToken()
    event.setView( "users/create" )
}
```

```html
<!-- View with manual token -->
<form method="post">
    <input type="hidden" name="_csrftoken" value="#prc.csrfToken#" />
    <!-- ...fields... -->
</form>
```

## Validating CSRF in Handlers

```boxlang
/**
 * handlers/Users.cfc
 */
class extends="coldbox.system.EventHandler" {

    // POST /users
    function store( event, rc, prc ) {
        // cbcsrf automatically validates on POST actions
        // If token is invalid it throws an exception

        // Manual validation if needed:
        if ( !verifyCSRFToken( rc._csrftoken ) ) {
            flash.put( "error", "Invalid security token. Please try again." )
            relocate( "users.create" )
        }

        userService.create( {
            name:  rc.name,
            email: rc.email
        } )

        flash.put( "success", "User created!" )
        relocate( "users.index" )
    }
}
```

## CSRF with AJAX Requests

```javascript
// Include CSRF token in AJAX requests via header
fetch('/users', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'X-CSRF-Token': document.querySelector('meta[name="csrf-token"]').content
    },
    body: JSON.stringify({ name: 'John', email: 'john@example.com' })
})
```

```html
<!-- Add CSRF token as meta tag in layout -->
<head>
    <meta name="csrf-token" content="#generateCSRFToken()#" />
</head>
```

## Excluding API Routes

```boxlang
moduleSettings = {
    cbcsrf: {
        // Exclude all API routes — APIs use JWT/API key auth instead
        exclude: [
            "^api\\..*",
            "^webhook\\..*"
        ]
    }
}
```

## Interceptor-Based Validation

```boxlang
/**
 * interceptors/CSRFInterceptor.cfc
 * Global CSRF validation for all POST requests
 */
class extends="coldbox.system.Interceptor" {

    function preProcess( event, interceptData ) {
        // Only check state-changing methods
        if ( !listContains( "POST,PUT,PATCH,DELETE", event.getHTTPMethod() ) ) {
            return
        }

        // Skip API routes (use JWT instead)
        if ( event.getCurrentEvent() startsWith "api." ) {
            return
        }

        // Validate token
        var token = event.getValue( "_csrftoken", "" )

        if ( !verifyCSRFToken( token ) ) {
            flash.put( "error", "Your session may have expired. Please try again." )
            relocate( event.getCurrentRoutedURL() )
        }
    }
}
```

## CSRF Token Helpers Reference

| Function | Description |
|----------|-------------|
| `csrf()` | Generate `<input type="hidden">` field with token |
| `generateCSRFToken()` | Return raw token string |
| `verifyCSRFToken( token )` | Validate a token string, returns boolean |

## Security Notes

- CSRF protection complements (doesn't replace) authentication
- API routes relying on JWT/API keys don't need CSRF tokens — exclude them
- Tokens are tied to the user's session
- `rotateTokens: true` is more secure but may break browser back-button behavior
- Always use HTTPS so tokens can't be intercepted
