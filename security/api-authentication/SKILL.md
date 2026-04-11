---
name: coldbox-security-api-authentication
description: "Use this skill when implementing API key authentication in ColdBox REST APIs, generating and validating API keys, caching API key lookups with CacheBox, implementing bearer token middleware, managing API key scopes and revocation, or adding an API key interceptor to protect REST endpoints."
---

# API Key Authentication in ColdBox

## Overview

API key authentication secures REST endpoints by requiring clients to send a secret key with each request. Keys are hashed in the database (SHA-256), cached on lookup for performance, and support scopes to restrict access.

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
install cbsecurity
```

## API Key Database Schema

```sql
CREATE TABLE api_keys (
    id          VARCHAR(36) PRIMARY KEY,
    user_id     VARCHAR(36) NOT NULL,
    name        VARCHAR(100),                  -- label e.g. "Production Backend"
    key_hash    VARCHAR(64) NOT NULL,           -- SHA-256 of the raw key (stored only)
    prefix      VARCHAR(10),                    -- readable prefix e.g. "sk_live_xxxx"
    scopes      VARCHAR(500) DEFAULT "*",       -- comma-delimited scope list
    revoked     TINYINT(1) DEFAULT 0,
    last_used   DATETIME,
    created_at  DATETIME
);
```

## API Key Service

```boxlang
/**
 * models/APIKeyService.cfc
 */
class singleton {

    property name="cacheBox" inject="cachebox"

    /**
     * Generate a new API key for a user.
     * Returns the raw key — only shown once.
     */
    function generate( required userID, name = "", scopes = "*" ) {
        var rawKey  = "sk_" & createUUID().replace( "-", "", "all" )
        var keyHash = hash( rawKey, "SHA-256" )
        var keyID   = createUUID()

        queryExecute(
            "INSERT INTO api_keys (id, user_id, name, key_hash, prefix, scopes)
             VALUES (:id, :userId, :name, :hash, :prefix, :scopes)",
            {
                id:     keyID,
                userId: arguments.userID,
                name:   arguments.name,
                hash:   keyHash,
                prefix: left( rawKey, 12 ),
                scopes: arguments.scopes
            }
        )

        return rawKey  // Return raw key once — not stored!
    }

    /**
     * Validate an incoming API key. Returns key record or false.
     * Caches validated keys for 5 minutes.
     */
    function validate( required key ) {
        var cache   = cacheBox.getCache( "default" )
        var cacheKey = "apikey_" & arguments.key

        // Check cache first
        var cached = cache.get( cacheKey )
        if ( !isNull( cached ) ) {
            return cached
        }

        // Hash and look up
        var keyHash = hash( arguments.key, "SHA-256" )
        var qry     = queryExecute(
            "SELECT * FROM api_keys WHERE key_hash = :hash AND revoked = 0",
            { hash: keyHash }
        )

        if ( !qry.recordCount ) {
            return false
        }

        var result = queryRowToStruct( qry, 1 )

        // Update last-used timestamp (async if possible)
        queryExecute(
            "UPDATE api_keys SET last_used = NOW() WHERE id = :id",
            { id: result.id }
        )

        // Cache the validated key
        cache.set( cacheKey, result, 5 )

        return result
    }

    /**
     * Check whether a key record has a given scope.
     */
    function hasScope( required struct keyRecord, required scope ) {
        if ( arguments.keyRecord.scopes == "*" ) return true
        return listFindNoCase( arguments.keyRecord.scopes, arguments.scope ) > 0
    }

    /**
     * Revoke an API key immediately (also bust cache).
     */
    function revoke( required keyID ) {
        queryExecute(
            "UPDATE api_keys SET revoked = 1 WHERE id = :id",
            { id: arguments.keyID }
        )

        // Clear any cached entry — iterate prefix patterns if needed
        cacheBox.getCache( "default" ).clearByKeySnippet( "apikey_" )
    }

    function listForUser( required userID ) {
        return queryExecute(
            "SELECT id, name, prefix, scopes, last_used, created_at
             FROM api_keys WHERE user_id = :userId AND revoked = 0
             ORDER BY created_at DESC",
            { userId: arguments.userID }
        )
    }
}
```

**CFML (`.cfc`):**

```cfml
/**
 * models/APIKeyService.cfc
 */
component {

    property name="cacheBox" inject="cachebox"

    /**
     * Generate a new API key for a user.
     * Returns the raw key — only shown once.
     */
    function generate( required userID, name = "", scopes = "*" ) {
        var rawKey  = "sk_" & createUUID().replace( "-", "", "all" )
        var keyHash = hash( rawKey, "SHA-256" )
        var keyID   = createUUID()

        queryExecute(
            "INSERT INTO api_keys (id, user_id, name, key_hash, prefix, scopes)
             VALUES (:id, :userId, :name, :hash, :prefix, :scopes)",
            {
                id:     keyID,
                userId: arguments.userID,
                name:   arguments.name,
                hash:   keyHash,
                prefix: left( rawKey, 12 ),
                scopes: arguments.scopes
            }
        )

        return rawKey  // Return raw key once — not stored!
    }

    /**
     * Validate an incoming API key. Returns key record or false.
     * Caches validated keys for 5 minutes.
     */
    function validate( required key ) {
        var cache   = cacheBox.getCache( "default" )
        var cacheKey = "apikey_" & arguments.key

        // Check cache first
        var cached = cache.get( cacheKey )
        if ( !isNull( cached ) ) {
            return cached
        }

        // Hash and look up
        var keyHash = hash( arguments.key, "SHA-256" )
        var qry     = queryExecute(
            "SELECT * FROM api_keys WHERE key_hash = :hash AND revoked = 0",
            { hash: keyHash }
        )

        if ( !qry.recordCount ) {
            return false
        }

        var result = queryRowToStruct( qry, 1 )

        // Update last-used timestamp (async if possible)
        queryExecute(
            "UPDATE api_keys SET last_used = NOW() WHERE id = :id",
            { id: result.id }
        )

        // Cache the validated key
        cache.set( cacheKey, result, 5 )

        return result
    }

    /**
     * Check whether a key record has a given scope.
     */
    function hasScope( required struct keyRecord, required scope ) {
        if ( arguments.keyRecord.scopes == "*" ) return true
        return listFindNoCase( arguments.keyRecord.scopes, arguments.scope ) > 0
    }

    /**
     * Revoke an API key immediately (also bust cache).
     */
    function revoke( required keyID ) {
        queryExecute(
            "UPDATE api_keys SET revoked = 1 WHERE id = :id",
            { id: arguments.keyID }
        )

        // Clear any cached entry — iterate prefix patterns if needed
        cacheBox.getCache( "default" ).clearByKeySnippet( "apikey_" )
    }

    function listForUser( required userID ) {
        return queryExecute(
            "SELECT id, name, prefix, scopes, last_used, created_at
             FROM api_keys WHERE user_id = :userId AND revoked = 0
             ORDER BY created_at DESC",
            { userId: arguments.userID }
        )
    }
}
```

## API Key Interceptor

```boxlang
/**
 * interceptors/APIKeyInterceptor.cfc
 * Validates API keys on every request to routes under /api/
 */
class {

    property name="apiKeyService" inject="APIKeyService"
    property name="logger"        inject="logbox:logger:{this}"

    function preProcess( event, interceptData, rc, prc, buffer ) {
        // Only enforce on /api/ routes
        if ( !event.getCurrentRoutedURL().startsWith( "/api/" ) ) {
            return
        }

        // Skip auth endpoint
        if ( event.getCurrentEvent() == "api.auth.generateKey" ) {
            return
        }

        // Extract key from Authorization header or query string
        var authHeader = getHTTPRequestData().headers[ "Authorization" ] ?: ""
        var rawKey     = ""

        if ( authHeader.startsWith( "Bearer " ) ) {
            rawKey = authHeader.removeFirst( "Bearer " ).trim()
        } else if ( !isNull( rc.api_key ) ) {
            rawKey = rc.api_key
        }

        if ( rawKey.isEmpty() ) {
            event.renderData( type = "json", data = { error: "API key required" }, statusCode = 401 )
            event.noExecution()
            return
        }

        var keyRecord = apiKeyService.validate( rawKey )
        if ( !keyRecord ) {
            logger.warn( "Invalid API key attempt from #CGI.REMOTE_ADDR#" )
            event.renderData( type = "json", data = { error: "Invalid or revoked API key" }, statusCode = 401 )
            event.noExecution()
            return
        }

        // Store key record in prc for downstream use
        prc.apiKeyRecord = keyRecord
        prc.apiUserID    = keyRecord.user_id
    }
}
```

**CFML (`.cfc`):**

```cfml
/**
 * interceptors/APIKeyInterceptor.cfc
 * Validates API keys on every request to routes under /api/
 */
component {

    property name="apiKeyService" inject="APIKeyService"
    property name="logger"        inject="logbox:logger:{this}"

    function preProcess( event, interceptData, rc, prc, buffer ) {
        // Only enforce on /api/ routes
        if ( !event.getCurrentRoutedURL().startsWith( "/api/" ) ) {
            return
        }

        // Skip auth endpoint
        if ( event.getCurrentEvent() == "api.auth.generateKey" ) {
            return
        }

        // Extract key from Authorization header or query string
        var authHeader = getHTTPRequestData().headers[ "Authorization" ] ?: ""
        var rawKey     = ""

        if ( authHeader.startsWith( "Bearer " ) ) {
            rawKey = authHeader.removeFirst( "Bearer " ).trim()
        } else if ( !isNull( rc.api_key ) ) {
            rawKey = rc.api_key
        }

        if ( rawKey.isEmpty() ) {
            event.renderData( type = "json", data = { error: "API key required" }, statusCode = 401 )
            event.noExecution()
            return
        }

        var keyRecord = apiKeyService.validate( rawKey )
        if ( !keyRecord ) {
            logger.warn( "Invalid API key attempt from #CGI.REMOTE_ADDR#" )
            event.renderData( type = "json", data = { error: "Invalid or revoked API key" }, statusCode = 401 )
            event.noExecution()
            return
        }

        // Store key record in prc for downstream use
        prc.apiKeyRecord = keyRecord
        prc.apiUserID    = keyRecord.user_id
    }
}
```

## Scope-Protected Handler

```boxlang
class extends="coldbox.system.RestHandler" {

    property name="apiKeyService" inject="APIKeyService"

    // POST /api/users
    function create( event, rc, prc ) {
        // Check scope
        if ( !apiKeyService.hasScope( prc.apiKeyRecord, "users.write" ) ) {
            return event.renderData(
                type       = "json",
                data       = { error: "Insufficient scope — requires users.write" },
                statusCode = 403
            )
        }

        var user = userService.create( rc )
        return event.renderData( type = "json", data = user, statusCode = 201 )
    }
}
```

**CFML (`.cfc`):**

```cfml
component extends="coldbox.system.RestHandler" {

    property name="apiKeyService" inject="APIKeyService"

    // POST /api/users
    function create( event, rc, prc ) {
        // Check scope
        if ( !apiKeyService.hasScope( prc.apiKeyRecord, "users.write" ) ) {
            return event.renderData(
                type       = "json",
                data       = { error: "Insufficient scope — requires users.write" },
                statusCode = 403
            )
        }

        var user = userService.create( rc )
        return event.renderData( type = "json", data = user, statusCode = 201 )
    }
}
```

## API Key Management Handler

```boxlang
// POST /api/keys/generate
function generate( event, rc, prc ) {
    var rawKey = apiKeyService.generate(
        userID = prc.currentUser.getID(),
        name   = rc.name ?: "Unnamed Key",
        scopes = rc.scopes ?: "read"
    )

    return event.renderData(
        type = "json",
        data = {
            key:    rawKey,
            notice: "Store this key securely — it will not be shown again."
        },
        statusCode = 201
    )
}

// DELETE /api/keys/:keyID
function revoke( event, rc, prc ) {
    apiKeyService.revoke( rc.keyID )
    return event.renderData( type = "json", data = { message: "Key revoked" } )
}

// GET /api/keys
function list( event, rc, prc ) {
    var keys = apiKeyService.listForUser( prc.currentUser.getID() )
    return event.renderData( type = "json", data = keys )
}
```

## Module Registration

```boxlang
// config/ColdBox.cfc — register interceptor
interceptors = [
    { class: "interceptors.APIKeyInterceptor", name: "APIKeyInterceptor" }
]
```

## Security Checklist

- [ ] Keys hashed with SHA-256 before storage — raw key never stored
- [ ] Raw key returned only once at generation time
- [ ] Validated keys cached (reduces DB hits)
- [ ] Cache cleared on revocation
- [ ] Scopes validated per endpoint
- [ ] Unauthorized attempts logged with IP
- [ ] HTTPS enforced in production (keys transmitted as Bearer tokens)
