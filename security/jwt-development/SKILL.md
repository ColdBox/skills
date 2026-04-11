---
name: coldbox-security-jwt
description: "Use this skill when implementing JWT (JSON Web Token) authentication in ColdBox REST APIs with CBSecurity, generating access/refresh tokens, validating bearer tokens, configuring JWT settings and secret keys, implementing token refresh endpoints, or securing API routes with JWT authentication middleware."
---

# JWT Development in ColdBox

## Overview

CBSecurity provides JWT (JSON Web Token) authentication for stateless REST APIs. JWT tokens encode user claims, are signed with a secret key, and validated on each request without server-side sessions.

## Installation

```bash
box install cbsecurity
box install jwtcfml
```

## JWT Configuration

```boxlang
// config/ColdBox.cfc
moduleSettings = {
    cbsecurity: {
        authenticationService: "JWTService@models",

        jwt: {
            issuer:    "myapp",
            audience:  "myapp-users",
            secretKey: getSystemSetting( "JWT_SECRET" ),
            expiration: 60,   // minutes

            refreshToken: {
                enabled:    true,
                expiration: 10080  // 7 days in minutes
            },

            tokenStorage: {
                enabled:  true,
                keyPrefix: "jwt_",
                provider: "CacheBox"
            }
        },

        firewall: {
            enabled: true,
            defaultAction: "block",
            statusCode: 401
        },

        rules: [
            // Public auth endpoints
            {
                whitelist: "api.auth.login,api.auth.register,api.auth.refresh",
                match: "event"
            },
            // All API routes require JWT
            {
                secureList: "^api\\.",
                match: "event",
                action: "block"
            }
        ]
    }
}
```

## JWT Service

```boxlang
/**
 * models/JWTService.cfc
 */
class singleton {

    property name="jwtService"  inject="JWTService@cbsecurity"
    property name="userService" inject="UserService"
    property name="bcrypt"      inject="@BCrypt"

    function generateToken( required user ) {
        return jwtService.encode( {
            sub:         user.id,
            email:       user.email,
            name:        user.name,
            roles:       user.roles,
            permissions: user.permissions,
            iat:         now(),
            exp:         dateAdd( "n", 60, now() )
        } )
    }

    function generateRefreshToken( required user ) {
        return jwtService.encode( {
            sub:  user.id,
            type: "refresh",
            iat:  now(),
            exp:  dateAdd( "d", 7, now() )
        } )
    }

    function authenticate( required username, required password ) {
        var user = userService.findByUsername( arguments.username )

        if ( !bcrypt.checkPassword( arguments.password, user.password ) ) {
            throw( type: "InvalidCredentials", message: "Invalid username or password" )
        }

        return {
            accessToken:  generateToken( user ),
            refreshToken: generateRefreshToken( user ),
            expiresIn:    3600,
            tokenType:    "Bearer"
        }
    }

    function refreshAccessToken( required refreshToken ) {
        var claims = jwtService.decode( arguments.refreshToken )

        if ( !claims.keyExists( "type" ) || claims.type != "refresh" ) {
            throw( type: "InvalidToken", message: "Not a refresh token" )
        }

        var user = userService.findById( claims.sub )
        return {
            accessToken: generateToken( user ),
            expiresIn:   3600,
            tokenType:   "Bearer"
        }
    }
}
```

## Auth Handler

```boxlang
/**
 * handlers/api/Auth.cfc
 */
class extends="coldbox.system.RestHandler" {

    property name="jwtService" inject="JWTService@models"

    // POST /api/auth/login
    function login( event, rc, prc ) {
        event.paramValue( "username", "" )
        event.paramValue( "password", "" )

        try {
            var tokens = jwtService.authenticate( rc.username, rc.password )
            event.getResponse()
                .setData( tokens )
                .setStatusCode( 200 )
        } catch ( InvalidCredentials e ) {
            event.getResponse()
                .setError( true )
                .addMessage( e.message )
                .setStatusCode( 401 )
        }
    }

    // POST /api/auth/refresh
    function refresh( event, rc, prc ) {
        event.paramValue( "refreshToken", "" )

        try {
            var result = jwtService.refreshAccessToken( rc.refreshToken )
            event.getResponse()
                .setData( result )
                .setStatusCode( 200 )
        } catch ( InvalidToken e ) {
            event.getResponse()
                .setError( true )
                .addMessage( "Invalid or expired refresh token" )
                .setStatusCode( 401 )
        }
    }
}
```

## Protected API Handler

```boxlang
/**
 * handlers/api/v1/Users.cfc
 * @secured — requires valid JWT
 */
class extends="coldbox.system.RestHandler" {

    property name="userService" inject="UserService"
    property name="cbsecurity"  inject="@cbsecurity"

    // GET /api/v1/users
    function index( event, rc, prc ) {
        event.paramValue( "page",    1 )
        event.paramValue( "perPage", 25 )

        prc.users = userService.list(
            page:    rc.page,
            perPage: rc.perPage
        )

        event.getResponse().setData( prc.users )
    }

    /**
     * Admin only
     * @secured admin
     */
    function destroy( event, rc, prc ) {
        userService.delete( rc.id )
        event.getResponse()
            .setData( {} )
            .setStatusCode( 204 )
    }
}
```

## Client Usage

```javascript
// Step 1: Login and get tokens
const response = await fetch('/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ username: 'user@example.com', password: 'pass' })
})
const { accessToken, refreshToken } = await response.json()

// Step 2: Use access token on subsequent requests
const data = await fetch('/api/v1/users', {
    headers: { 'Authorization': `Bearer ${accessToken}` }
})

// Step 3: Refresh when expired
const refreshResponse = await fetch('/api/auth/refresh', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ refreshToken })
})
```

## JWT Security Checklist

- Store `JWT_SECRET` in environment variable, never in source code
- Use sufficiently long secret (256+ bits)
- Set short expiration for access tokens (15-60 minutes)
- Use refresh tokens for long-lived sessions
- Validate issuer and audience claims
- Implement token revocation via token storage when needed
- Use HTTPS — JWT payloads are base64-encoded, not encrypted
