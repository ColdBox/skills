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
        enabled          = true,
        tokenKey         = "_token",                    // form field / header key
        rotateTokens     = false,                       // rotate on every request
        tokenExpiration  = 0,                           // 0 = session lifetime (minutes)
        verifyReferer    = true,
        storageStrategy  = "session",                   // session | cookie
        protectedMethods = [ "POST", "PUT", "PATCH", "DELETE" ],
        exemptions       = [],                          // regex patterns to skip

        // Custom handler on invalid token (optional)
        onInvalidToken   = function( event, rc, prc ) {
            throw( type = "InvalidCSRFToken", message = "Invalid or missing CSRF token" )
        }
    }
}
```

## Core Helpers

Available as mixins in handlers, views, and layouts:

| Helper | Returns | Purpose |
|--------|---------|---------|
| `csrfToken()` | string | Current CSRF token |
| `csrf()` | HTML string | `<input type="hidden" name="_token" value="...">` |

## Production Patterns

### HTML Form Protection

```cfml
<!--- Preferred: use the csrf() helper to inject the hidden field --->
<form method="POST" action="#event.buildLink('user.update')#">
    #csrf()#
    <input type="text" name="username">
    <button type="submit">Update</button>
</form>

<!--- Or manually --->
<input type="hidden" name="_token" value="#csrfToken()#">
```

### AJAX / Fetch Requests

```html
<!--- Embed token in <head> for JavaScript access --->
<meta name="csrf-token" content="#csrfToken()#">
```

```js
// Vanilla fetch
fetch( '/api/user/update', {
    method  : 'POST',
    headers : {
        'X-CSRF-TOKEN' : document.querySelector( 'meta[name="csrf-token"]' ).content,
        'Content-Type' : 'application/json'
    },
    body : JSON.stringify( data )
} )

// Axios — set globally once
axios.defaults.headers.common['X-CSRF-TOKEN'] =
    document.querySelector( 'meta[name="csrf-token"]' ).content
```

### Manual Validation in a Handler

```js
class MyHandler extends coldbox.system.EventHandler {
    @inject("CSRFService@cbcsrf")
    property name="csrfService";

    function save( event, rc, prc ) {
        if ( !csrfService.verify( rc._token ?: "" ) ) {
            throw( type = "InvalidCSRFToken", message = "CSRF token validation failed" )
        }
        // process ...
    }
}
```

### Route Exemptions

```js
// config/Router.cfc — exempt webhooks from CSRF
route( "/webhooks/stripe" )
    .withHandler( "webhooks.stripe" )
    .withAction( { POST : "process" } )
    .exemptFromCSRF()
```

Or via config regex pattern:

```js
exemptions = [ "^api/webhooks/", "^public/payments/" ]
```

### Token Management API

```js
property name="csrfService" inject="CSRFService@cbcsrf";

var token    = csrfService.getToken()          // current token
var newToken = csrfService.generateToken()     // generate new token
var isValid  = csrfService.verify( token )     // validate
csrfService.rotateToken()                      // force rotation
```

## Best Practices

- **Include `csrf()` in every state-changing form** — POST, PUT, PATCH, DELETE
- **Exempt read-only APIs** (`GET`, `HEAD`, `OPTIONS`) — they're protected by design
- **Exempt webhooks by route** — not by disabling CSRF entirely
- **Use `X-CSRF-TOKEN` header for AJAX** rather than body param in JSON APIs
- **Never log CSRF tokens** — treat them like short-lived secrets
- **Set `rotateTokens = true`** for higher-security applications to limit token reuse
- **Do not exempt login forms** — they should also include CSRF tokens

## Documentation

- cbcsrf: https://github.com/coldbox-modules/cbcsrf
