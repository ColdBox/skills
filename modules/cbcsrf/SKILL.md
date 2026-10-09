---
name: cbcsrf
description: >
  Use this skill when adding CSRF protection to ColdBox/BoxLang applications with the cbcsrf module.
  Covers token generation, form helpers, AJAX/meta-tag patterns, manual handler validation, route exemptions,
  SPA integration, token rotation, and configuration best practices for preventing cross-site request forgery.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# CBCSRF Skill

## When to Use This Skill

Load this skill when:
- Protecting HTML forms from CSRF attacks
- Sending CSRF tokens in AJAX/fetch/axios requests
- Exempting webhook or public API endpoints from CSRF validation
- Configuring token rotation, expiration, or storage strategy
- Building SPAs that need a global token header
- Manually validating tokens in handler actions

## Language Mode Reference

| Concept | BoxLang (`.bx`) | CFML (`.cfc`) |
|---------|-----------------|---------------|
| Mixin helper | `csrfToken()`, `csrf()` | same — available in handlers/views |

## Installation & Configuration

```bash
box install cbcsrf
```

```js
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

## Core Helpers

Mixins available in handlers, views, and layouts:

| Helper | Returns | Purpose |
|--------|---------|---------|
| `csrfToken( key, forceNew )` | string | Generate (or reuse) a token for the key |
| `csrfVerify( token, key )` | boolean | Validate a token for the key |
| `csrf( key, forceNew )` | HTML string | Hidden `<input name="csrf" id="csrf">` with a token |
| `csrfField( key, forceNew )` | HTML string | Same hidden field, plus JS that reloads the page when the token expires (uses `rotationTimeout`) |
| `csrfRotate()` | this | Clear all stored tokens |

The same operations exist on the service, WireBox ID `@cbcsrf`: `generate( key, forceNew )`, `verify( token, key )`, `rotate()`.

## How Verification Works

Verification is not automatic. Set `enableAutoVerifier = true` to load the `VerifyCsrf@cbcsrf` interceptor, or verify manually.
The interceptor:

- Skips `GET`, `HEAD` and `OPTIONS`
- Skips events matching a `verifyExcludes` regex
- Skips actions annotated with `skipCsrf`
- Reads the token from the `csrf` request value or the `x-csrf-token` header
- Throws `TokenNotFoundException` when no token is sent and `TokenMismatchException` when it is invalid

## Production Patterns

### HTML Form Protection

```cfml
<form method="POST" action="#event.buildLink( 'user.update' )#">
    #csrf()#
    <input type="text" name="username">
    <button type="submit">Update</button>
</form>
```

### AJAX / Fetch Requests

```html
<meta name="csrf-token" content="#csrfToken()#">
```

```js
fetch( '/api/user/update', {
    method  : 'POST',
    headers : {
        'x-csrf-token' : document.querySelector( 'meta[name="csrf-token"]' ).content,
        'Content-Type' : 'application/json'
    },
    body : JSON.stringify( data )
} )
```

### Manual Validation in a Handler

```cfml
component {

    function save( event, rc, prc ){
        if ( !csrfVerify( rc.csrf ?: "" ) ) {
            throw( type = "TokenMismatchException", message = "CSRF token validation failed" )
        }
        // process ...
    }

}
```

### Skipping Verification

```cfml
// Skip one action when the auto verifier is on
function webhook( event, rc, prc ) skipCsrf {
    // ...
}
```

Or by event regex in the settings:

```js
verifyExcludes = [ "^api\\..*", "^webhooks\\..*" ]
```

### Token Endpoint for SPAs

Set `enableEndpoint = true` and request `GET /cbcsrf/generate/:key?`. The endpoint returns a token for the key (default `default`).
It is a `secured` handler, so authenticated users only when cbsecurity or cbguard is installed, and it returns `404` when disabled.

## Best Practices

- **Include `csrf()` in every state-changing form**: POST, PUT, PATCH, DELETE
- **Use the `x-csrf-token` header for AJAX** rather than a body param in JSON APIs
- **Exclude webhooks and token-authenticated APIs** with `verifyExcludes` or `skipCsrf`, not by disabling CSRF entirely
- **Never log CSRF tokens**: treat them like short-lived secrets
- **Lower `rotationTimeout`** for higher-security applications to limit token reuse
- **Do not exempt login forms**: they should also include CSRF tokens

## Documentation

- cbcsrf: https://github.com/coldbox-modules/cbcsrf
