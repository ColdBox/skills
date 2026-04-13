---
name: cbstorages
description: >
  Use this skill when persisting data across requests in ColdBox/BoxLang using cbstorages. Covers
  Session, Cookie, Cache, Request, and Application storage adapters; configuration; get/set/exists/
  delete/clear operations; encryption; TTLs; and patterns for shopping carts, user preferences,
  and API token caching.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# CBStorages Skill

## When to Use This Skill

Load this skill when:
- Storing user preferences, shopping cart data, or session state
- Persisting cross-request data with configurable backends (session, cookie, cache, database)
- Encrypting sensitive values before storage
- Swapping storage backends without changing application code
- Managing cookie expiry, HTTP-only flags, and secure attributes

## Installation

```bash
box install cbstorages
```

## Configuration

### config/modules/cbstorages.cfc

```js
function configure() {
    return {
        // Encrypt stored values (uses CFML encrypt() with AES/256)
        encryption : {
            enabled   : false,                              // turn on for sensitive data
            algorithm : "AES/CBC/PKCS5Padding",
            key       : getSystemSetting( "STORAGE_KEY", "" )
        },

        // Cookie storage defaults
        cookie : {
            domain   : "",
            secure   : true,    // HTTPS only
            httpOnly : true,    // not accessible via JS
            samesite : "Strict"
        }
    }
}
```

## Storage Adapters

### Injection

```js
property name="sessionStorage"     inject="SessionStorage@cbstorages";
property name="cookieStorage"      inject="CookieStorage@cbstorages";
property name="cacheStorage"       inject="CacheStorage@cbstorages";
property name="requestStorage"     inject="RequestStorage@cbstorages";
property name="applicationStorage" inject="ApplicationStorage@cbstorages";
```

### Universal API (All Adapters)

```js
// Set
sessionStorage.set( "cart", cartData )
sessionStorage.set( "cart", cartData, 30 )  // TTL in minutes (where supported)

// Get
var cart = sessionStorage.get( "cart" )
var cart = sessionStorage.get( "cart", {} )  // with default

// Check existence
if ( sessionStorage.exists( "cart" ) ) { ... }

// Delete
sessionStorage.delete( "cart" )

// Clear all keys in this storage
sessionStorage.clear()

// Get all keys
var keys = sessionStorage.getKeys()
```

### Cookie Storage (Extra Options)

```js
// Set with expiry and attributes
cookieStorage.set(
    name     = "theme",
    value    = "dark",
    expires  = 365,         // days
    secure   = true,
    httpOnly = true,
    sameSite = "Lax"
)
```

## Production Patterns

### Shopping Cart (Session-Backed)

```js
class CartService {

    property name="sessionStorage" inject="SessionStorage@cbstorages";

    private string function cartKey() {
        return "cart_" & security.getCurrentUser().getId()
    }

    struct function getCart() {
        return sessionStorage.get( cartKey(), { items: [], total: 0 } )
    }

    void function addItem( productId, qty = 1, price ) {
        var cart  = getCart()
        var items = cart.items ?: []
        var found = false

        for ( var item in items ) {
            if ( item.productId == productId ) {
                item.qty += qty
                found = true
                break
            }
        }

        if ( !found ) {
            items.append( { productId: productId, qty: qty, price: price } )
        }

        cart.items = items
        cart.total = items.reduce( ( acc, item ) => acc + ( item.qty * item.price ), 0 )
        sessionStorage.set( cartKey(), cart )
    }

    void function clearCart() {
        sessionStorage.delete( cartKey() )
    }
}
```

### User Preference Storage (Cookie-Backed)

```js
// Store UI preferences in encrypted cookie
cookieStorage.set(
    name     = "prefs",
    value    = serializeJSON( { theme: "dark", pageSize: 25 } ),
    expires  = 365,
    secure   = true,
    httpOnly = false    // Must be JS-readable for theme switching
)

// Retrieve
var prefs = deserializeJSON( cookieStorage.get( "prefs", '{"theme":"light","pageSize":10}' ) )
```

### API Token Caching (Cache-Backed)

```js
class ExternalApiService {

    property name="cacheStorage" inject="CacheStorage@cbstorages";
    property name="hyper"        inject="HyperBuilder@hyper";

    string function getAccessToken() {
        var cacheKey = "external_api_token"

        if ( cacheStorage.exists( cacheKey ) ) {
            return cacheStorage.get( cacheKey )
        }

        var response = hyper.post( "https://auth.api.example.com/oauth/token", {
            grant_type    : "client_credentials",
            client_id     : getSystemSetting( "API_CLIENT_ID" ),
            client_secret : getSystemSetting( "API_CLIENT_SECRET" )
        } )

        var token = response.json().access_token

        // Cache for 50 minutes (token valid 60 min)
        cacheStorage.set( cacheKey, token, 50 )

        return token
    }
}
```

## Best Practices

- **Enable encryption** for sensitive data (tokens, PII) in cookie or cache storage
- **Store secrets in environment variables**, not hardcoded in config
- **Use `httpOnly = true`** for all session/auth cookies to prevent XSS access
- **Use `secure = true`** for all cookies in production (HTTPS)
- **Use `samesite = "Strict"` or `"Lax"`** to prevent CSRF via cookies
- **Use cache storage over session** for data that should survive server restarts
- **Never store passwords or raw credentials** in any cbstorage adapter
- **Scope session keys per user** when multiple users share a server-side session store

## Documentation

- cbstorages: https://github.com/coldbox-modules/cbstorages
